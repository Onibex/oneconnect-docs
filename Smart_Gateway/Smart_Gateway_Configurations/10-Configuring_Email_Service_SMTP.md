# Smart Gateway Configuration Manual

## Configuring the Email Service (SMTP)

The platform uses a dedicated email service to send account-related emails (user activation, password resets) and log alert notifications. Before these features work, the email service must be configured with your SMTP server credentials.

> 📌 **Prerequisite:** You need an SMTP server (such as ZeptoMail, SendGrid, Gmail SMTP, or any corporate mail server) and its credentials.

---

## Variables to Configure

The following environment variables must be added to the email builder deployment in Kubernetes:

| Kubernetes Variable | Description |
|---|---|---|
| `EMAIL_SOURCEADDRESS` | Sender name and email that appears in the "From" field | 
| `EMAIL_SMTPHOST` | Hostname of your SMTP server |
| `EMAIL_SMTPPORT` | Port used by the SMTP server |
| `EMAIL_SMTPAUTH` | Whether authentication is required (`true`/`false`) |
| `EMAIL_SMTPSTARTTLSENABLE` | Whether STARTTLS is enabled (`true`/`false`) |
| `EMAIL_SMTPFROM` | The sender email address |
| `EMAIL_SMTPSSLPROTOCOLS` | SSL/TLS protocol version |
| `EMAIL_USER` | SMTP username or API key |
| `EMAIL_PASSWORD` | SMTP password or API secret |

---

## Kubernetes YAML Example

Add these variables to your email builder deployment:

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
  - name: EMAIL_PASSWORD
    value: "emailpassword"
```

After applying the configuration, redeploy the email builder service.