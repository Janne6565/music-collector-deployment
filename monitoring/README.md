# SigNoz dashboards

The version-controlled copy of Rekordo's two dashboards. SigNoz keeps dashboards in its
own database, not in Kubernetes manifests, so **ArgoCD does not apply these** — they are
here so a dashboard someone edits in the UI can be diffed, reviewed and restored.

| File | Dashboard | What it answers |
|---|---|---|
| `dashboard-rekordo-collection-community.json` | Rekordo — Collection & Community | Who signed up, what they collect, what they are hunting for |
| `dashboard-rekordo-service-health.json` | Rekordo — Service Health | Whether it works, where it is slow, and what it is quietly throwing away |

Both take an `env` variable (`prod` / `staging`), which selects on the
`deployment.environment` resource attribute set per overlay.

## Re-exporting after a UI edit

    curl -s -H "SIGNOZ-API-KEY: $KEY" \
      https://signoz.jannekeipert.de/api/v1/dashboards/<uuid> \
      | python3 -c 'import json,sys; d=json.load(sys.stdin)["data"]; print(json.dumps({"uuid":d["id"],"data":d["data"]}, indent=2, sort_keys=True))' \
      > monitoring/dashboard-<slug>.json

The uuid is the `uuid` field in each file. The API key lives in the login keychain
(`security find-generic-password -s signoz.jannekeipert.de -a alert-rule-automation -w`);
never commit it.

## Why the panels are PromQL and not the query builder

Rekordo's metrics arrive over OTLP, so SigNoz stores them under their dotted names
(`rekordo.users.total`, `http.server.request.duration.count`). PromQL reaches those through
the `{__name__="..."}` form, and dotted *label* names need quoting the same way
(`{"deployment.environment"="prod"}`). It also matches how the other services' dashboards
here are written.

A panel counting bad things ends in `or vector(0)` on purpose: without it a healthy service
matches no series and the panel reads "no data", which looks identical to a panel that is
broken.

## Alert rules

Not here — the rules live in `cluster-deployment/infrastructure/signoz-alerts/`, beside
every other service's, because they are managed as one set and delivered through one n8n
channel.
