# Smart Gateway Configuration Manual

## Configuring the Email Service (SMTP)

The platform uses a dedicated email service to send account-related emails (user activation, password resets) and log alert notifications. Before these features work, the email service must be configured with your SMTP server credentials.

> 📌 **Prerequisite:** You need an SMTP server (such as ZeptoMail, SendGrid, Gmail SMTP, or any corporate mail server) and its credentials.

---

## Variables to Configure

The following environment variables must be added to the email builder deployment in Kubernetes:

| Kubernetes Variable | Description | Example |
|---|---|---|
| `EMAIL_SOURCEADDRESS` | Sender name and email that appears in the "From" field | `Onibex One Connect <contact@onibex.com>` |
| `EMAIL_SMTPHOST` | Hostname of your SMTP server | `smtp.zeptomail.com` |
| `EMAIL_SMTPPORT` | Port used by the SMTP server | `587` |
| `EMAIL_SMTPAUTH` | Whether authentication is required (`true`/`false`) | `true` |
| `EMAIL_SMTPSTARTTLSENABLE` | Whether STARTTLS is enabled (`true`/`false`) | `true` |
| `EMAIL_SMTPFROM` | The sender email address | `noreply@yourcompany.com` |
| `EMAIL_SMTPSSLPROTOCOLS` | SSL/TLS protocol version | `TLSv1.2` |
| `EMAIL_USER` | SMTP username or API key | `emailapikey` |
| `EMAIL_PASSWORD` | SMTP password or API secret | `wSsVR60g+x/yCv0...` |

---

## Kubernetes YAML Example

Add these variables to your email builder deployment:

```yaml
env:
  - name: EMAIL_SOURCEADDRESS
    value: "Onibex One Connect <contact@onibex.com>"
  - name: EMAIL_SMTPHOST
    value: "smtp.zeptomail.com"
  - name: EMAIL_SMTPPORT
    value: "587"
  - name: EMAIL_SMTPAUTH
    value: "true"
  - name: EMAIL_SMTPSTARTTLSENABLE
    value: "true"
  - name: EMAIL_SMTPFROM
    value: "noreply@yourcompany.com"
  - name: EMAIL_SMTPSSLPROTOCOLS
    value: "TLSv1.2"
  - name: EMAIL_USER
    value: "emailapikey"
  - name: EMAIL_PASSWORD
    value: "your-smtp-password-or-api-key"
```

After applying the configuration, redeploy the email builder service.