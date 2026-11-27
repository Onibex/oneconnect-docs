# Configuring the Email Service (SMTP)

The platform uses a dedicated email service to send account-related emails (user activation, password resets) and log alert notifications. Before these features work, the email service must be configured with your SMTP server credentials.

> 📌 **Prerequisite:** You need an SMTP server (such as ZeptoMail, SendGrid, Gmail SMTP, or any corporate mail server) and its credentials.

---

## Variables to Configure

The following environment variables must be added to the email builder deployment in Kubernetes:

| Kubernetes Variable | Description |
|---|---|
| `EMAIL_SOURCEADDRESS` | Sender name and email that appears in the "From" field |
| `EMAIL_SMTPHOST` | Hostname of your SMTP server |
| `EMAIL_SMTPPORT` | Port used by the SMTP server |
| `EMAIL_SMTPAUTH` | Whether authentication is required (`true`/`false`) |
| `EMAIL_SMTPSTARTTLSENABLE` | Whether STARTTLS is enabled (`true`/`false`) |
| `EMAIL_SMTPFROM` | The sender email address |
| `EMAIL_SMTPSSLPROTOCOLS` | SSL/TLS protocol version |
| `EMAIL_USER` | SMTP username or API key |
| `EMAIL_PASSWORD` | SMTP password or API secret. Set by the Helm chart from `appSecrets.emailPassword`; do not write it in the deployment (see below) |

---

## Kubernetes YAML Example

### The SMTP password

The Helm chart stores the SMTP password in a Kubernetes Secret and passes it to the email builder as `EMAIL_PASSWORD`. Set it in your `secrets.local.yaml` (the same file used at install, see the [AWS](../AWS/SmartGateway_Helm_Chart_AWS.md) or [Azure](../Azure/SmartGateway_Helm_Chart_Azure.md) guide) and run `helm upgrade` with your usual values files:

```yaml
appSecrets:
  emailPassword: "<SMTP password or API key>"
```

If you manage the Secrets yourself (`appSecrets.existingSecret`), put the password under the `email-password` key of that Secret.

### The other SMTP settings

Add the remaining variables to your email builder deployment:

```yaml
env:
  - name: EMAIL_SOURCEADDRESS
    value: "Email Source address <contact@example.com>"
  - name: EMAIL_SMTPHOST
    value: "smtp.host.com"
  - name: EMAIL_SMTPPORT
    value: "SMTPORT"
  - name: EMAIL_SMTPAUTH
    value: "true"
  - name: EMAIL_SMTPSTARTTLSENABLE
    value: "true"
  - name: EMAIL_SMTPFROM
    value: "noreply@yourcompany.com"
  - name: EMAIL_SMTPSSLPROTOCOLS
    value: "TLSv1.2"
  - name: EMAIL_USER
    value: "emailuser"
  # EMAIL_PASSWORD is already set by the chart from appSecrets.emailPassword
```

After applying the configuration, redeploy the email builder service.
