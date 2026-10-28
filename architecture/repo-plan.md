# EV_PnC_Consent_ repository family

**Project:** EV Owners Roaming Consent (`../../roaming-consent/`) · **Date:** 2026-10-25 · **Status:** plan; repos not created yet
**Inputs:** consent spec (31 requirements), HLD containers, infrastructure BOM (`../../national-hub-service/06-planlama-ve-yatirim.md` §3), AWS Well-Architected Framework (used as a cloud-neutral review lens; hosting stays in Türkiye), Soft Skills and Business (allocation and org-design rules)

This family **replaces the single `EV_PNC_roaming-consent` repo** planned earlier the same day. `EV_PNC_consent` (e-Devlet vehicle-ownership consent) stays as it is; the two meet through events.

## 1. How the boundaries were drawn

A repo is a unit of **ownership, release cadence and security boundary**. It is not a unit of scaling: one repo can ship several deployables that scale independently. So we split a repo only when one of these is true:

| Rule | Why | Where it applies |
|---|---|---|
| Different load profile and SLO on a hot path | Fault isolation, independent scaling (REL 10, PERF 1) | `decision` (p95 ≤ 50 ms, 1,000 rps) vs `registry` |
| Separation of duties | No component may both write grants and rewrite the audit trail (SEC 3) | `audit` |
| Different reviewers or audience | Legal approves texts; SRE owns runbooks (OPS 2) | `notices`, `ops` |
| Different exposure | Infrastructure topology and runbooks are private (SEC 1, SEC 5) | `infra`, `ops` private |
| Shared interface for many consumers | One source of truth for OpenAPI/AsyncAPI and SDKs, contract tests | `contracts` |

Everything else stays together as modules of a modular monolith (`registry`) until a trigger in §5 fires. This follows the "lead bullets, not silver bullets" rule: a 2–3 person team should not run a dozen microservices on day one.

## 2. Repository list

| # | Repository | Wave | Visibility | Owner | Purpose | Requirements |
|---|---|---|---|---|---|---|
| 1 | `EV_PnC_Consent_docs` | 1 | public | Architect + DPO | ADRs, PRD/HLD, specs, legal-basis matrix, threat model, Well-Architected review log | all |
| 2 | `EV_PnC_Consent_contracts` | 1 | public | Tech lead | OpenAPI, AsyncAPI, JSON Schema (receipt per ISO/IEC TS 27560), generated SDKs, contract tests | GATE, REV, REC |
| 3 | `EV_PnC_Consent_registry` | 1 | public | Backend A | Catalogue, notices, grants, receipts, safeguards, rights, tenancy; also ships the propagation worker | CAT, GRANT, REC, REV, RIGHTS, LEGAL, TEN |
| 4 | `EV_PnC_Consent_decision` | 1 | public | Backend B | Roaming decision point: allow/deny, fail closed, event-fed cache | GATE-002..003, NFR-001 |
| 5 | `EV_PnC_Consent_infra` | 1 | **private** | Platform/SRE | Terraform, Helm, Argo CD per BOM; two Türkiye DCs; cost tags; reuses `EV_PNC_infra` modules | NFR-002 |
| 6 | `EV_PnC_Consent_testkit` | 1 | public | QA | Gherkin acceptance, revocation-SLA test, PDP load test, DAST | verification |
| 7 | `EV_PnC_Consent_audit` | 2 | public | Backend B + DPO | Hash-chained audit trail, WORM anchoring, DPO query | AUD-001..002 |
| 8 | `EV_PnC_Consent_ops` | 2 | **private** | Platform/SRE | SLOs as code, dashboards, alerts with owners, runbooks, playbooks, game days | REV-002, NFR-001 |
| 9 | `EV_PnC_Consent_notices` | 2 | public | DPO + UX writing | Notice texts TR/EN/DE/NL with hashes; legal review by pull request | CAT-003, REC-003 |
| 10 | `EV_PnC_Consent_dpo-console` | 2 | public | Frontend | DPO and support console: inventory, receipts, rights, safeguards | RIGHTS, LEGAL, AUD-001 |
| 11 | `EV_PnC_Consent_edevlet-bridge` | 3 | public | Backend A | FAPI 2.0 client, PAR, mTLS, SSF receiver for e-Devlet `roaming.share` | EDV-001..002 |
| 12 | `EV_PnC_Consent_tenant-portal` | 3 | public | Product + Frontend | Onboarding, branding, credentials and metering for other eMSP brands | TEN-001..002 |

**Waves**

- **Wave 1, now (Phase 0–1, Nov 2026 – Feb 2027):** 6 repos, enough to build and test the core.
- **Wave 2, before the pilot (by Mar 2027):** `audit` is split out of the registry, and ops, notices and the console are added.
- **Wave 3, on trigger:** `edevlet-bridge` when the e-Devlet technical workshop starts; `tenant-portal` when the first external eMSP signs.

**Not created on purpose (reuse what exists):** mobile consent screens live in `EV_PNC_mobile`; hub mocks come from `EV_PNC_simulator`; Terraform modules come from `EV_PNC_infra`; the key vault is shared with the platform (TR-18) through a separate partition.

## 3. BOM → repository mapping and sizing

Sizing is an inference (I) from the consent NFRs. Pilot = under 5,000 drivers, one tenant. National = about 1 million drivers, several tenants.

| BOM item | Owning repo (IaC) | Used by | Pilot | National |
|---|---|---|---|---|
| Kubernetes namespace / node pool | `infra` | registry, decision, audit, propagation | Shared EV_PNC cluster, dedicated namespace, ~16 vCPU total | Dedicated node pool, HPA, ~60–80 vCPU per DC |
| PostgreSQL HA (registry) | `infra` | registry | 3 × 4 vCPU, 16 GB, 200 GB | 3 × 16 vCPU, 64 GB, 2 TB + read replicas; partition by tenant |
| PostgreSQL (audit, partitioned) | `infra` | audit | Same cluster, own database | Own cluster; ~0.7 TB/year (≈2 M decisions/day × 500 B, ×2 for indexes) |
| Redis (PDP cache) | `infra` | decision | 3 × 2 vCPU, 4 GB | 3 × 4 vCPU, 16 GB per DC |
| Kafka topics and ACLs | `infra` | registry, decision, propagation, ocpi-emsp | Topics on the shared EV_PNC Kafka, 6 partitions | Dedicated topics, 24 partitions, keyed by driver pseudonym |
| Object storage with WORM lock | `infra` | audit (daily root hash), receipts archive | < 10 GB/year | ~50 GB/year |
| API gateway + WAF + mTLS | `infra` | all APIs | Shared EV_PNC gateway, own routes | Own gateway pair per DC |
| HSM / KMS (JWS signing keys, mTLS keys) | `infra` | registry, audit, edevlet-bridge | Partition in the platform key vault | Dedicated HSM partition, yearly rotation |
| Observability (metrics, logs, traces) | `ops` (rules), `infra` (stack) | all | Shared stack | Shared stack, consent SLO dashboards |
| CI runners, artifact registry | `infra` | all | Shared | Shared |

Two Türkiye data centres (active + warm standby) for national scale; RPO ≤ 5 min, RTO ≤ 1 h. In the pilot, a single DC with tested backups is enough. This is a deliberate trade of reliability against cost for the pilot; security and operations are not traded.

## 4. Well-Architected review of the repo plan

| Pillar | What the plan does | Questions |
|---|---|---|
| Operational excellence | One named owner per repo; runbooks and alerts as code in `ops`; every alert has an owner and a runbook; small, reversible releases per deployable | OPS 2, OPS 5–7, OPS 10 |
| Security | `audit` separated (separation of duties); keys only in HSM/KMS; `infra` and `ops` private; DAST and dependency scans in `testkit`; threat model in `docs` | SEC 1, SEC 3, SEC 7–9, SEC 11 |
| Reliability | PDP isolated and fail closed; propagation scaled on Kafka lag; DR via `infra`; revocation storm and DC failover game days | REL 5, REL 9–13 |
| Performance efficiency | Hot read path (PDP) on an event-fed cache, measured at p95/p99 with automated load tests | PERF 1, PERF 3, PERF 5 |
| Cost optimization | Wave plan avoids idle repos and environments; mandatory tags `project=consent, component, env, owner`; preprod scales to zero outside test windows | COST 2–4, COST 9 |
| Sustainability | Audit data tiered to cold storage after 13 months; no always-on load-test environment | SUS 2, SUS 4 |

Review cadence: a light review before each wave and after any significant change, logged in `docs`.

## 5. Scaling triggers (when to split further)

| Trigger | Action |
|---|---|
| More than 3 hub integrations, or propagation needs its own on-call | Move the propagation worker from `registry` to `EV_PnC_Consent_propagation` |
| A separate team owns rights requests, or more than [500] requests per month | Move rights to `EV_PnC_Consent_rights` |
| Second tenant signs | Partition registry tables by tenant; create `tenant-portal` (wave 3) |
| PDP p95 above 50 ms at peak for 2 weeks | Add read replicas per DC; consider a sidecar cache in `EV_PNC_ocpi-emsp` |
| Consent analytics outgrow ops dashboards | Create `EV_PnC_Consent_analytics` (pseudonymous event pipeline) |

## 6. Conventions

- Names: `EV_PnC_Consent_<kebab-case>`. No `__`.
- Every repo: `CODEOWNERS`, protected `main`, required checks (build, tests, contract tests, secret scan), conventional commits, semver releases.
- Public repos: Apache-2.0. Private repos: no licence file.
- No personal data, signed contracts or secrets in any repo. Safeguard records live in the registry database.
- Specs stay traceable: requirement IDs `RC-*` in code tests and pull request titles.

## 7. How to create them

Repository creation from this cloud session returns 403, so the repos have to be created from your machine:

```bash
cd repo-plan/consent
gh auth login            # once
WAVE=1 ./create_consent_repos.sh     # 6 repos; WAVE=2, WAVE=3 or WAVE=all later
```

The script skips repos that already exist, creates public repos with Apache-2.0 and private ones without a licence, and pushes the starter READMEs from `readmes/`.
