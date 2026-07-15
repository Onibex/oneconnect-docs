# Smart Gateway Configuration Manual

## Enabling Email Alerts for Logs

The logs module can send email notifications when `ERROR` or `FATAL` logs are detected. By default, this feature is **disabled**. To enable it, configure the following settings.

> 📌 **Prerequisite:** The [Email Service (SMTP)](10-Configuring_Email_Service_SMTP.md) must be configured first.

---

## Variables to Configure

The following environment variables must be added to the **oneconnect-logs** deployment in Kubernetes:

| Kubernetes Variable | Description | Default | Required |
|---|---|---|---|
| `LOGEMAILSENDER_ENABLE` | Enables email sending for log alerts | `false` | Yes, set to `true` |

> ℹ️ The email cache prevents duplicate alerts: the same error type in the same workspace will only trigger one email every **60 minutes**.

---

## Kubernetes YAML Example

Add these variables to your **oneconnect-logs** deployment:

```yaml
env:
  - name: LOGEMAILSENDER_ENABLE
    value: "true"
```

---

## Frontend Configuration

The **"Add Email"** button in the Logs dashboard is also hidden by default. To make it visible to users, set the following environment variable in the **frontendoneconnect** deployment:

```yaml
env:
  - name: NEXT_PUBLIC_ENABLE_ADD_EMAIL_LOGS_BUTTON
    value: "true"
```

When this is enabled, users can manage which email addresses receive log alerts from their SAP Links.

---

## Summary

To fully enable log email alerts, configure **all three** deployments:

| Deployment | Variable | Value |
|---|---|---|
| `emailbuilder` | `EMAIL_*` | SMTP server credentials |
| `oneconnect-logs` | `LOGEMAILSENDER_ENABLE` | `true` |
| `oneconnect-logs` | `TARGET_URL_EMAIL` | Email builder URL |
| `frontendoneconnect` | `NEXT_PUBLIC_SMTP_ENABLED` | `true` |
