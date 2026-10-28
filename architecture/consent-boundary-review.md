# Consent sınırı incelemesi: kendi kayıt defterimiz mi, devlet tarafı merkezi servis mi?

**Tarih:** 25 Ekim 2026 · **Soru (Steerfocus):** consent için ayrı repo/repolar olmalı değil mi; e-Devlet veya EPDK gibi farklı bir kuruluş için yapılmış merkezi bir consent management olmalı değil mi?
**Yöntem:** `book-skills:architecture-review` prosedürü, bağlam/sınır adımında `microservices-and-ddd` (stratejik DDD, bağlam haritası), paketleme ilkeleri (`agile-ppp-packaging-principles`: CCP, CRP, ADP, SDP, SAP). Etiketler: **F** kaynaklı olgu, **I** çıkarım, **U** bilinmiyor.

## 1. Özet ve öneri

1. **Evet, ayrı repolar olmalı ve zaten ayrı:** 12 repolu `EV_PnC_Consent_` ailesi `EV_PNC_` setinden ayrıdır; bağımlılık tek yönlüdür (EV_PNC → `Consent_contracts`).
2. **Ama "consent" tek bir şey değil, üç katman.** Üçünü karıştırmak sınırı bozuyordu:
   - **Katman 1, bizim sistem kaydımız:** KVKK veri sorumlusu olarak rıza/tercih kanıtı, güvence kaydı, roaming karar noktası, denetim izi. Bunu devlete bırakamayız.
   - **Katman 2, devletin merkezi servisi:** e-Devlet Mobility Consent Service (C3). Vatandaş kendi onaylarını tek yerden görür. Sahibi ve işletmecisi e-Devlet'tir (DDO politikası, Türksat işletimi, **F**: `national-hub-service/01-PRD.md`). Biz inşa edip işletmeyiz; sözleşmeyi (kapsamlar, imzalı onay) ve açık kaynak bir referans/mock sağlarız.
   - **Katman 3, köprü:** `Consent_edevlet-bridge`. Katman 2 varsa onu kayıt defterimize bir **rıza kaynağı** olarak bağlar.
3. **EPDK'nın consent rolü yok.** EPDK C1 (şarj veri erişim noktası) ve C2 (TR kimlik kaydı) için adaydır; bunlar istasyon verisi ve kimlik tahsisidir, kişisel rıza değildir.
4. **Öneri: hibrit.** Katman 1 sistem kaydı olarak kalır (pilot için tek zorunlu yol, e-Devlet'e bağımlı değil); Katman 2 isteğe bağlı üst kaynak olarak tasarlanır.
5. **Repo planı değişikliği:** `EV_PNC_national-services` tek repo olarak kalırsa farklı kurumların (EPDK ve e-Devlet) sahip olacağı kodu bir arada tutar. Kuruma göre ikiye bölündü: **`EV_PNC_national-consent`** (C3, e-Devlet tarafı) ve **`EV_PNC_national-data`** (C1, C2, C4, EPDK tarafı). Plan 18 yerine 19 `EV_PNC_` repo (+12 `EV_PnC_Consent_`), ikisi de dalga 3.

## 2. Sürücüler ve ölçülebilir nitelikler (sıralı)

| # | Nitelik | Ölçü | Neden bu sırada |
|---|---|---|---|
| 1 | Hesap verebilirlik | Denetimde makbuzu/güvence kaydı olmayan aktarım = 0 | Veri sorumlusu biziz; kanıtı elimizde tutmak zorundayız (I, hukuk teyidi gerekir) |
| 2 | Karar yolu: erişilebilirlik ve gecikme | PDP p95 ≤ 50 ms, hata halinde kapalı (RC-GATE) | Her token push ve yetkilendirme buradan geçer |
| 3 | Pilot takvimi bağımsızlığı | e-Devlet olmadan canlı | e-Devlet takvimi 12–24 ay (U), NHS-ADR-05 pilot bağımsızlığını kabul etti, F |
| 4 | Geri alma | Hub'da token geçersiz ≤ 15 dk, %99 | RC-REV-001 |
| 5 | Tarafsızlık | İkinci eMSP kiracı olarak katılabilir | Faz 3 hedefi |
| 6 | Kapsam ve işletme yükü | 2–3 kişilik ekip, kamu hizmeti işletmiyoruz | `N-08` ile sınırlanan enabler hattı |

## 3. Bağlam ve sınırlar (DDD)

- **Alt alanlar.** Bizim çekirdek alanımız navigasyon + Plug&Charge + misafir deneyimi. Rıza, bizim için *destekleyici*; ama alt parçaları farklı: (a) rıza toplama ve makbuz **genel** bir alan, devletin servisi bunu bir gün üstlenebilir; (b) güvence kaydı (standart sözleşme, bildirim tarihleri) ve roaming karar noktası **bizim kontrolör yükümlülüğümüze ve sıcak yolumuza özgü**, dışarı verilemez.
- **Bağlam haritası.** `Consent_registry`/`decision` bizim bağlamımız; EV_PNC servislerine **Open Host Service + Published Language** (`consent-contracts`). e-Devlet MCS **yukarı akış**; biz küçük müşteri olduğumuz için Customer-Supplier değil **Conformist + Anticorruption Layer**: tüm MCS dili `edevlet-bridge` içinde kalır, kayıt defteri içine sızmaz.
- **Alan modelinde karşılığı:** `RoamingGrant.channel` zaten `app` veya `e-Devlet` (RC HLD); yani kaynak ayrımı modelde var. Eklenen: bunu açık bir **ConsentSource portu** yapmak (K-17).
- **KVKK notu (I, hukuk teyidi şart):** roaming aktarımının ana dayanağı standart sözleşme; uygulamadaki ekran çoğunlukla tercih + bilgilendirme. Bu yüzden MCS'nin `roaming.share` kapsamı hukuki yükü kaldırmaz, yalnız sürücüye tek yerden kontrol ve bizim için ek bir kanıt kaynağı verir. MCS'nin asıl değeri araç sahipliği (`vehicle.ownership.verify`, şu an askıda) ve `contract.binding.verify`.

## 4. Seçenekler (puan 1–5, ağırlıklı; puanlar yargıdır, I)

| Nitelik (ağırlık) | A. Yalnız devlet MCS | B. Yalnız kendi kayıt defterimiz | **C. Hibrit** | D. Devlet için biz inşa ve işletelim |
|---|---|---|---|---|
| Hesap verebilirlik (5) | 2 | 5 | **5** | 3 |
| Karar yolu (5) | 1 | 5 | **5** | 3 |
| Pilot bağımsızlığı (4) | 1 | 5 | **5** | 2 |
| Geri alma (4) | 3 | 4 | **4** | 3 |
| Tarafsızlık (3) | 4 | 3 | **4** | 5 |
| Kapsam ve işletme yükü (4) | 4 | 3 | **4** | 1 |
| **Toplam / 125** | 59 | 107 | **114** | 69 |

- **A reddedildi:** MCS yok, SLA bilinmiyor ve karar yolu dış servise bağlanır; kontrolör kanıtı yine bizde kalır.
- **B yedek:** sorunsuz ama devletin tek yerden kontrol hedefiyle bağlantı kurmaz ve e-Devlet gelirse yeniden yazım ister.
- **D reddedildi:** kamu hizmeti işletmek hem kapsamımız dışında hem satın alma süreçleri nedeniyle takvim dışı; yalnız referans uygulama açık kaynak verilir.
- **Bedeli (C):** iki model ve bir köprü bakımı; MCS gelmezse köprü kullanılmaz (zaten dalga 3, askıda).

## 5. Hedef mimari ve repo değişiklikleri

```
EV_PNC_* servisleri ──► EV_PnC_Consent_contracts (karar SDK, olaylar)        [kararlı, soyut]
                              ▲
EV_PnC_Consent_registry ──► ConsentSource portu ◄── app kanalı (varsayılan)
                                              ◄── EV_PnC_Consent_edevlet-bridge ──► mcs-contracts
EV_PNC_national-consent  (C3 referansı + mock)  ──► trust-contracts ◄── EV_PNC_national-data (C1, C2, C4)
```

| Değişiklik | Gerekçe (ilke) |
|---|---|
| `national-services` → `national-consent` + `national-data` | CCP: farklı sahipler ve yayın döngüleri. CRP: C3 kullanıcıları C1 ve C2'yi almak zorunda kalmaz. |
| İki repo arasında yalnız `trust-contracts` kenarı | SAP/SDP: bağımlılık soyut, I = 0 bileşene. İkisi de C4'ü benimserse `EV_PNC_national-trust` ayrılır. |
| `edevlet-bridge` yalnız `mcs-contracts`'a bağlı | ADP: dönüş kenarı yok; mock bir görüntü olarak çekilir, kütüphane olarak değil. |
| `Consent_registry` içinde ConsentSource portu | DIP: kayıt defteri e-Devlet'i bilmez. |

`tools/packaging_check.py --strict`: 33 bileşen, 19 repo, döngü yok, SDP ihlali yok.

## 6. Veri ve tutarlılık

- **Tek yazar:** her rıza `RoamingGrant` olarak yalnız kayıt defterinde yazılır; MCS'den gelen onay bir *olay*dır (SSF), kayıt defteri onu idempotent işleyip kendi makbuzunu üretir (at-least-once varsayımı).
- **Karar noktası** yalnız kayıt defterinin olay beslemeli önbelleğini okur; MCS kapalıyken app kanalı çalışır, MCS'den gelen geri alma gecikirse ≤ 15 dk hedefi MCS tarafı için de SSF sözleşmesine yazılır (NHS-MCS-003).
- **Çakışma:** aynı sürücünün app'te "evet", MCS'de "hayır" demesi durumunda **en kısıtlayıcı olan kazanır** (fail closed).

## 7. Riskler ve uygunluk fonksiyonları

| Risk | Karşılık | Fonksiyon (nerede çalışır) |
|---|---|---|
| MCS hiç gelmez | App kanalı kalıcı çözüm olarak tasarlı | MCS mock kapalıyken karar testi geçer, p95 ≤ 50 ms (testkit, CI) |
| Köprü kayıt defterine sızar | ACL | `packaging_check`: yalnız `edevlet-bridge` `mcs-contracts`'a bağlı olabilir (CI) |
| İki kaynak çelişir | En kısıtlayıcı kazanır | Çakışma senaryosu Gherkin'de (testkit) |
| Geri alma MCS'den gecikir | SSF sözleşmesi ≤ 15 dk | Geri alma ölçümü (preprod, game day) |
| Referans uygulama işletme yükü olur | Biz işletmeyiz; `N-08` sınırı | Kapasite payı raporu (haftalık) |

## 8. ADR-0007 (öneri; kabul K-15 kartında)

**Karar.** `EV_PnC_Consent_` ailesi, kontrolör kanıtı ve roaming karar noktası için sistem kaydıdır. Devlet MCS'si isteğe bağlı bir üst rıza kaynağıdır ve yalnız `edevlet-bridge` üzerinden bağlanır. Referans uygulama `EV_PNC_national-consent` içinde açık kaynak olarak yayımlanır; işletim e-Devlet'indir.
**Sonuçlar.** (+) pilot e-Devlet'ten bağımsız, karar yolu dışa bağlı değil. (−) iki model ve bir köprü bakımı; MCS gelince sürücünün iki yerde onayı görünür, çakışma kuralı gerekir.
**Reddedilenler.** Yalnız devlet; devlet için işletme; EPDK'yı consent sahibi yapmak (rolü yok).

## 9. Açık sorular

- Kontrolör kanıt yükümlülüğünün MCS imzalı onayıyla ne ölçüde karşılanabileceği: hukuk (TR-05, K-01).
- e-Devlet'in `roaming.share` kapsamını kabul edip etmeyeceği ve takvimi: teknik çalıştay (U).
- C4'ün sahibi: EPDK mi, e-Devlet mi, ortak mı (PRD'de ikisi birden yazılı).
