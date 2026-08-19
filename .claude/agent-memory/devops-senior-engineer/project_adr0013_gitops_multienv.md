---
name: project-adr0013-gitops-multienv
description: ADR-0013 (base/ + staging/ overlay) implementation — kustomize/helm quirks discovered, staging naming/port conventions, ArgoCD staging Application
metadata:
  type: project
---

ADR-0013 (GitOps + config multi-ambiente) implemented 2026-08-10. Full detail
in `docs/implementation/IMPL-ADR-0013-2026-08-10.md` (workspace root, not this
repo). Companion session `IMPL-ADR-0014-0020-2026-08-10.md` did the IAM/OIDC
half (site/ stack roles) and explicitly left the staging Ingress to this ADR.

## Structure
- `base/` — only `service-account.yml`, `analysis-template.yml`,
  `secrets/mongo-certificate.yml` (+ kustomization.yml). Deliberately NOT
  `namespace.yml` (Namespace kind isn't affected by namePrefix/nameSuffix —
  confirmed empirically — sharing one would render the same literal name in
  both overlays), NOT `config-map.yml` (inherently per-environment), NOT
  kyverno policies (see below).
- `production/` — unchanged behavior (verified byte-identical kustomize
  build diff except the two intentional changes below). Now includes `../base`.
- `staging/` — new overlay, `namePrefix: dpe-stg-`, `nameSuffix: -stg`,
  `namespace: staging`. Reuses production's Helm chart templates via
  `helmGlobals.chartHome: ../production/application` (no template
  duplication) but has its OWN full `values/*.yaml` per app (see kustomize
  bug below) and its own raw manifests for everything else (configmap,
  secret-store, external-secret, certificate, network-policy, resource-quota,
  limit-range, seed-job, ingress, pdb).

## kustomize/helm gotchas discovered (kustomize v5.8.1, `--enable-helm`)

1. **`helmChart.valuesInline` + `valuesMerge: merge` silently does nothing.**
   Tried to avoid duplicating values.yaml by declaring only the staging delta
   inline over `valuesFile: ../production/application/<app>/values.yaml`. The
   valuesFile always won with zero error/warning. Pivoted to a full
   `staging/values/<app>.yaml` per app (copy of prod's values.yaml with env
   fields overridden) — this is the reliable, well-established pattern, use
   it going forward instead of re-attempting valuesInline.
2. **`commonLabels` transformer pollutes `NetworkPolicy` peer
   `podSelector.matchLabels`** (not just `metadata.labels`). Confirmed this
   ALSO affects the pre-existing `production/infrastructure/network-policies/
   egress.yml` DNS rule (harmless there only because a `0.0.0.0/0` no-port
   ipBlock rule already allows everything). Workaround: use
   `namespaceSelector.matchLabels` (e.g. `kubernetes.io/metadata.name: kube-system`)
   instead of `podSelector.matchLabels` for cross-cutting allow rules —
   `namespaceSelector.matchLabels` is NOT polluted by commonLabels.
3. **`PodDisruptionBudget.spec.selector.matchLabels` is NOT polluted** by
   commonLabels (unlike NetworkPolicy peers) — safe to use bare `app: <name>`.
4. **Namespace kind names are immune to `namePrefix`/`nameSuffix`** — confirmed
   via `kustomize build`. Don't rely on it for anything meant to differ per
   overlay.
5. **`ClusterPolicy` (kyverno) names ARE prefixed** despite being
   cluster-scoped — if you share kyverno policies via `base/` across two
   overlays you get TWO ClusterPolicy objects (different names, same body),
   which is redundant if both list the same `match.namespaces`. Decision:
   kyverno stays production-only for now (not extended to staging in this
   ADR), documented as a scope trim, not a technical blocker.
6. `helmGlobals.chartHome` pointing outside the kustomization root
   (`../production/application`) works fine WITHOUT
   `--load-restrictor LoadRestrictionsNone` — only direct `resources:`/
   `valuesFile:` entries outside the root trigger that sandbox error. No
   change needed to `argocd/argocd-cm-patch.yml` (still just
   `kustomize.buildOptions: "--enable-helm"`).

## Architecture finding — escalated, not fixed here

`production/infrastructure/network-policies/ingress.yml` and `egress.yml`
(ADR-0004) both contain an `ipBlock: 0.0.0.0/0` rule with NO port
restriction. Kubernetes NetworkPolicy is additive allow-list only — this
blanket rule means the new `deny-staging-ingress.yml` guardrail (ADR-0013
Decision 4, dpe side) is NOT actually enforcing anything while that rule
exists. Real enforcement of dpe↔staging isolation currently comes from
staging's OWN default-deny egress (staging never allows egress toward dpe —
that direction IS fully enforced). Flagged for an ADR-0004 amendment
(scope the ipBlock rule to a real ALB/VPC CIDR) — not touched in this
session per minimal-risk-to-prod instruction.

## Staging naming/port conventions (for future ADR-0012/0014 work)

- nodePort offset: staging = production + 100 (order 30000→30100,
  order-preview 30001→30101, main 30002→30102, invoice-generator
  30003→30103, identity-server 30004→30104, health-checker 30005→30105,
  notificator 30006→30106, main-preview 30007→30107, identity-server-preview
  30008→30108).
- Resources: staging requests 64Mi/50m, limits 128Mi/100m uniformly (smaller
  than every production value, including order's 256Mi/256m).
- Secrets Manager key used by staging ExternalSecret is a PLACEHOLDER
  (`devopsproject/app/secrets-stg`) — confirm real name against ADR-0012 once
  the `serverless` stack -stg resources are actually applied.
- `dpe-stg-ecr-image-pull-credentials-stg` is referenced in every staging
  Helm values file but NOT provisioned by this repo (same as prod's
  `dpe-ecr-image-pull-credentials-prod` — provisioned by out-of-band
  automation, location unknown/undocumented in this repo).

## 3 hardcodes found and parameterized in shared Helm charts (zero prod behavior change, verified by diff)

- `main/order/identity-server` rollout.yml: `prePromotionAnalysis.templates[0].
  templateName` was a literal `dpe-health-check-prod` string, not
  `.Values`-driven. Now `.Values.<app>Deployment.blueGreen.analysisTemplateName`.
- `invoice-generator` scaledobject.yml: `queueURL` was a literal SQS URL. Now
  `.Values.invoiceGeneratorDeployment.autoScaling.keda.queueURL`.
- `notificator` scaledobject.yml: same pattern for `EmailNotificationQueue`.

If a 4th environment or a chart refactor happens later, grep templates/*.yml
for any other literal `dpe-...-prod` / hardcoded ARN/URL strings before
assuming `.Values` already covers everything — the values.yaml files looked
fully parameterized but 3 real hardcodes were hiding in templates.
