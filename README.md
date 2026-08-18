# hyperdx-observability

Auto-troubleshooting for HyperDX/ClickStack alerts on Krateo. The assembly/installer unit for the
feature: it declares alerts as Kubernetes `Alert` CRs, reconciles them into HyperDX, and — when an
alert fires — runs an **Autopilot root-cause analysis** captured as a `TroubleshootingReport` CR
that the portal Observability page surfaces.

## Why a reconciler and not KOG

ClickStack **v2.27** exposes alerts + webhooks only through its **passport session-authenticated**
internal API (`/api/alerts`, `/api/webhooks`) — there is no bearer/apikey REST API. KOG
(oasgen-provider) authenticates with a static bearer token and can't drive a login/session, so it
can't manage these resources. Instead a small **session-driven reconciler** (folded into the
`krateo-alert-troubleshooter` image) logs in and calls the internal API. See the retired
`../hyperdx-alerts-kog/RETIRED.md`.

## The loop

```
Alert CR (observability.krateo.io)
   └─ reconciler (session) ──> HyperDX: webhook (with a Handlebars body!) + dashboard-tile + alert
                                   └─ alert fires ──> POST webhook ──> handler /webhook
                                                          └─ Autopilot A2A RCA ──> TroubleshootingReport CR
   reconciler also mirrors live alert state (OK/ALERT) back into Alert .status
```

Notes learned the hard way (baked into the image):
- HyperDX registration needs a policy-compliant password (>=12, upper+lower+digit+special) + a
  matching `confirmPassword`. The reconciler self-registers the admin on a fresh HyperDX.
- The session cookie is scoped `Domain=localhost`; it's sent explicitly, not via the jar.
- A generic webhook **must** carry a `body` (compiled as Handlebars); a body-less webhook makes
  every notification fail silently.
- Alerts use a **dashboard tile** source (there is no saved-search REST route in v2.27).

## Install

### As a Krateo composition (intended)
1. Package + publish the chart: `helm package chart && helm push hyperdx-observability-0.1.0.tgz oci://ghcr.io/krateo-blueprints/charts`
2. Apply `compositiondefinition.example.yaml` (edit the chart URL/version), then create the
   `HyperdxObservability` instance CR. Krateo renders the chart server-side (so `lookup` works and
   the admin password is generated + persisted).

### Directly with Helm (dev)
```
helm install hyperdx-observability ./chart -n krateo-system \
  --set hyperdx.adminPassword='Aa1!<something-strong>'   # set explicitly: plain `helm template` has no cluster for lookup
```

## The "if" (feature toggles — values.yaml)
| value | effect |
|---|---|
| `installCRDs` | install the Alert + TroubleshootingReport CRDs |
| `troubleshooter.enabled` | run the webhook receiver + Autopilot handler |
| `reconciler.enabled` / `reconciler.interval` | reconcile Alert CRs -> HyperDX + mirror status |
| `defaultAlerts.enabled` + `errorLogVolume`/`warningLogVolume` | ship the default log-volume alerts |
| `hyperdx.adminPassword` | pin the admin password (else auto-generated + kept) |

## Authoring alerts
Create `Alert` CRs (or rely on the defaults):
```yaml
apiVersion: observability.krateo.io/v1alpha1
kind: Alert
metadata: { name: my-alert, namespace: krateo-system }
spec:
  displayName: "My alert"
  where: "SeverityText:error"     # lucene over the logs source; empty = all
  interval: "5m"                  # 1m|5m|15m|30m|1h|6h|12h|1d
  threshold: 5
  thresholdType: above            # above|below|...
  message: "Autopilot will triage."
```
`kubectl get alerts -n krateo-system` shows each alert's mirrored `STATE`; `kubectl get tsr`
lists the troubleshooting reports.
