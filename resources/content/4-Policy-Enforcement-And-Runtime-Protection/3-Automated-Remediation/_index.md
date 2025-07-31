---
title: "Automated Response to Violations with Falcosidekick"
date: 2025-07-15
weight: 3
chapter: false
pre: " 4.3 "
---

## Main Content

* [Introduction to Falcosidekick](#introduction-to-falcosidekick)
* [Deploying Falcosidekick with Falco](#deploying-falcosidekick-with-falco)
* [Configuring Alert Notifications](#configuring-alert-notifications)
* [Testing Alerts](#testing-alerts)

---

## Introduction to Falcosidekick

**Falcosidekick** is a service that supports Falco in sending alerts to external systems such as:

* Slack, Discord
* Email
* Webhook, Kafka, NATS
* Prometheus, Grafana, Elasticsearch...

Falcosidekick receives logs from Falco and forwards them according to configuration.

---

## Deploying Falcosidekick with Falco

### Step 1: Install Falco with Falcosidekick

```bash
helm install falcosidekick falcosecurity/falcosidekick --namespace falco --set config.slack.webhookurl="https://hooks.slack.com/services/xxx"
```

🔁 You can replace Slack with other webhook URLs as needed.

### Step 2: Check running pods

```bash
kubectl get pods -n falco
```

✅ Expected result: 2 pods running: `falco` and `falcosidekick`

---

## Configuring Alert Notifications

You can make additional edits in the `values.yaml` file if you want to configure multiple alert systems.

Example: Send to Discord, Prometheus, Webhook...

```yaml
falcosidekick:
  config:
    discord:
      webhookurl: "https://discord.com/api/webhooks/xxx"
    webhook:
      address: "http://your-api-endpoint"
```

Then upgrade the chart:

```bash
helm upgrade falco falcosecurity/falco -n falco -f values.yaml
```

---

## Testing Alerts

### Generate anomalous events

```bash
kubectl run -i --tty attacker --image=alpine -- sh
```

Then run:

```bash
touch /etc/passwd
```

### Check sent alerts

* Check Falcosidekick logs:

```bash
kubectl logs -l app=falcosidekick -n falco
```

* View notifications on Slack/Discord/Webhook according to configuration.

📸 *Take a screenshot of the alert received on Slack or Discord to include in documentation.*

---

## Notes

* Falcosidekick helps integrate Falco with various alerting systems.
* Can be extended to send alerts to SIEM, automation systems (XDR, SOAR).

