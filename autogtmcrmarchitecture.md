# AutoGTM → Full CRM + Marketing Pipeline: Architecture Recommendation

**Prepared for:** Salil
**Date:** August 10, 2026
**Context:** Extend the existing AutoGTM lead-gen platform — which already has its own email sending engine — with a proper CRM, campaign/tracking mechanism, and follow-through pipeline, self-hosted on Kubernetes. (Earlier drafts of this doc assumed Instantly.ai stayed in the loop; that's no longer the case — AutoGTM's own sender replaces it, and this doc now focuses purely on the CRM + marketing automation layer sitting downstream of it.)

---

## 1. Recommendation

**SuiteCRM (CRM/pipeline) + Mautic (marketing automation), both self-hosted on your Kubernetes cluster, wired to AutoGTM's own sender engine via API.**

This combo directly answers what you said you're missing: SuiteCRM gives you the actual CRM (contact/lead records, deal stages, ownership, activity history — the "pipeline" you asked for), and Mautic gives you the campaign/tracking/follow-through mechanism (open & click tracking, multi-step nurture sequences, lead scoring, segmentation) that a raw sending engine doesn't provide on its own.

This beats the EspoCRM + Mautic combo suggested earlier for one concrete reason: SuiteCRM ships an **official, first-party Mautic integration module** ([SuiteCRM Mautic Integration docs](https://store.suitecrm.com/docs/mautic-integration/introduction), [8.x user guide](https://store.suitecrm.com/docs/mautic-integration/user-guide-suitecrm-8-x)) that does bidirectional contact/lead sync, campaign membership sync, and lead-score sync out of the box. EspoCRM has no native Mautic connector — you'd be stuck routing through a third-party automation tool like Make or Integrately, which is an extra paid dependency and a weaker link for something this central to your pipeline.

SuiteCRM also has a maintained, official **Bitnami Helm chart** ([artifacthub.io/packages/helm/bitnami/suitecrm](https://artifacthub.io/packages/helm/bitnami/suitecrm)) and Bitnami Docker image ([hub.docker.com/r/bitnami/suitecrm](https://hub.docker.com/r/bitnami/suitecrm)), so it drops cleanly into a k8s-native deploy process instead of requiring you to hand-roll manifests.

Mautic has multiple community Helm chart / k8s distribution options:

- [FacetInteractive/mautic-k8s](https://github.com/FacetInteractive/mautic-k8s) — purpose-built HA/scalable Mautic-on-k8s distribution with Helm charts ([announcement post](https://facetinteractive.com/blog/announcing-mautic-helm-charts-k8s-distribution))
- [audacioustux/mautic-chart](https://github.com/audacioustux/mautic-chart) — Helm chart + container image
- Step-by-step manual guide if you want full control: [kozlov.ski/mautic-5x-kubernetes-setup](https://kozlov.ski/mautic-5x-kubernetes-setup/)

Both are PHP/MySQL-based (SuiteCRM: PHP + MySQL/MariaDB; Mautic: PHP + MySQL/MariaDB, optionally + Elasticsearch for advanced segmentation), which keeps your ops surface consistent — one DB engine, one runtime, one set of PHP-FPM tuning knobs to learn instead of two.

---

## 2. Where this fits with your current stack

AutoGTM already has its own sending engine for the first-touch outreach — that piece is solved and out of scope here. What it's missing is everything downstream: a real CRM to track contacts/deals through a pipeline, and a campaign/tracking layer to run structured follow-up sequences instead of one-off sends. That's exactly the gap SuiteCRM + Mautic fills.

### Data flow

```
AutoGTM's sender engine (first-touch send + reply/engagement capture)
        │  webhook or polling: send/open/reply/bounce events
        ▼
AutoGTM (lead-gen logic, qualification rules — unchanged)
        │  REST API: push qualified lead as SuiteCRM Lead/Contact
        ▼
SuiteCRM (system of record: pipeline, deal stage, ownership, activities)
        │  native Mautic connector: contact + segment sync
        ▼
Mautic (marketing automation: follow-up/nurture sequences, open & click
         tracking, lead scoring, landing pages, segmentation, drip campaigns)
        │  engagement events (opens/clicks/score changes) sync back
        ▼
SuiteCRM (your team sees enriched, scored leads in one pipeline view)
```

In practice: your sender engine handles the first-touch email. The moment AutoGTM sees a reply or qualifying signal, it creates/updates a record in SuiteCRM via its REST API. SuiteCRM's Mautic module keeps that contact's marketing status (nurture stage, score, campaign membership) in sync automatically, so your team works out of one pipeline view while Mautic runs the structured follow-up sequences, tracking, scoring, and re-engagement campaigns in the background. You can also route Mautic's own sending through your existing sender engine/mailboxes if you want a single sending identity end-to-end — that's a config choice, not an architectural blocker.

---

## 3. Deployment plan (Kubernetes)

1. **Namespace + secrets**: dedicate a namespace (e.g. `crm`) with its own MySQL/MariaDB instance (or a shared instance with separate schemas for SuiteCRM and Mautic — separate schemas is safer for backup/restore granularity).
2. **SuiteCRM**: deploy via the Bitnami Helm chart. Set persistent volume claims for uploads, configure SMTP for transactional mail, enable the REST API (v8/legacy v4.1 depending on your integration needs).
3. **Mautic**: deploy via a chosen Helm chart (FacetInteractive's distribution is purpose-built for HA if you expect meaningful send/segment volume; audacioustux's is lighter-weight for a single-instance start). Configure a mail-sending transport for Mautic's own nurture emails — either a dedicated SMTP/API relay, or your existing sender engine's mailboxes if you want one unified sending identity. Either way, make sure Mautic's compliance features (unsubscribe headers, list-management, suppression lists) are switched on, since nurture email has different legal/deliverability requirements than first-touch cold outreach.
4. **Install the SuiteCRM Mautic Integration module** ([store.suitecrm.com/addons/mautic-integration](https://store.suitecrm.com/addons/mautic-integration)) and configure the connection using Mautic's OAuth2 API credentials.
5. **Ingress + TLS**: expose both behind your existing ingress controller with cert-manager, on subdomains like `crm.yourdomain.com` and `automation.yourdomain.com`.
6. **AutoGTM ↔ SuiteCRM wiring**: build a small integration service (or a job/webhook receiver inside AutoGTM) that listens for your sender engine's events (sent/opened/replied/bounced) and calls the SuiteCRM REST API to upsert Leads/Contacts and log Activities. This is custom glue code regardless of which CRM you pick, since it's specific to your own sender engine's API — budget dev time here.
7. **Backups**: since this becomes your system of record for pipeline + marketing history, put both databases on the same backup cadence/retention policy as your other production data.

---

## 4. Why not the other options

- **EspoCRM + Mautic**: EspoCRM has a cleaner UI, but the CRM↔automation link relies on third-party iPaaS tools (Make, Integrately) rather than a native module — more moving parts, another vendor dependency, and sync logic you don't fully control.
- **Odoo (all-in-one)**: consolidates CRM + marketing automation + email under one app, which is operationally simpler, but you lose the "best-of-breed" flexibility you asked for, and Odoo's marketing automation is noticeably less mature than Mautic's for complex nurture logic.
- **Twenty + Mautic/n8n**: Twenty is a great modern API-first CRM if you want to build a lot of custom tooling around it, but it's young, has no native marketing-automation connector yet, and would need the most custom integration work of any option here — reasonable if you want maximum control and have engineering bandwidth to spare, riskier if you want to move fast.

---

## 5. Next steps checklist

- [ ] Stand up MySQL/MariaDB in the `crm` namespace (or confirm existing DB service to reuse)
- [ ] Deploy SuiteCRM via Bitnami Helm chart, verify REST API access
- [ ] Deploy Mautic via chosen Helm chart, configure dedicated SMTP/API sending transport
- [ ] Install + configure SuiteCRM's native Mautic Integration module
- [ ] Build the AutoGTM → SuiteCRM webhook/API bridge for your sender engine's send/open/reply/bounce events
- [ ] Define lead qualification rules: what triggers a lead moving from "raw sender-engine send" into "SuiteCRM pipeline + Mautic nurture"
- [ ] Decide whether Mautic sends nurture email through its own transport or through your existing sender engine's mailboxes
- [ ] Set up Mautic nurture sequences, lead scoring rules, and segments
- [ ] Configure backup/retention policy for both databases

---

## Sources

- [SuiteCRM Mautic Integration – Introduction](https://store.suitecrm.com/docs/mautic-integration/introduction)
- [SuiteCRM Mautic Integration – User Guide (8.x)](https://store.suitecrm.com/docs/mautic-integration/user-guide-suitecrm-8-x)
- [SuiteCRM Mautic Integration – User Guide (7.x)](https://store.suitecrm.com/docs/mautic-integration/user-guide-suitecrm-7-x)
- [SuiteCRM Mautic Integration Module (store listing)](https://store.suitecrm.com/addons/mautic-integration)
- [Sync Data Between SuiteCRM and Mautic (blog)](https://store.suitecrm.com/blog/sync-data-between-suitecrm-and-mautics-marketing-automation-platform)
- [Bitnami SuiteCRM Helm chart (Artifact Hub)](https://artifacthub.io/packages/helm/bitnami/suitecrm)
- [Bitnami SuiteCRM Docker image](https://hub.docker.com/r/bitnami/suitecrm)
- [FacetInteractive/mautic-k8s (Helm charts, HA Mautic on k8s)](https://github.com/FacetInteractive/mautic-k8s)
- [Announcing Mautic Helm Charts & K8s Distribution](https://facetinteractive.com/blog/announcing-mautic-helm-charts-k8s-distribution)
- [audacioustux/mautic-chart](https://github.com/audacioustux/mautic-chart)
- [How to Set Up Mautic 5.x on Kubernetes (step-by-step)](https://kozlov.ski/mautic-5x-kubernetes-setup/)
- [EspoCRM Docker installation docs](https://docs.espocrm.com/administration/docker/installation/) (for reference/comparison)
- [EspoCRM Docker Hub image](https://hub.docker.com/r/espocrm/espocrm/) (for reference/comparison)
