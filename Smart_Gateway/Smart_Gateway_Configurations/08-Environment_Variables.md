# Smart Gateway Configuration Manual

## Web Application Environment Variables Reference

The OneConnect Smart Gateway web application uses environment variables to control certain features and behaviors. This document describes the available configuration options.

---

## SMTP-Requiring Actions: `NEXT_PUBLIC_SMTP_ENABLED`

| Variable | Type | Default | Description |
|---|---|---|---|
| `NEXT_PUBLIC_SMTP_ENABLED` | `"true"` or `"false"` | `"true"` | Controls features that require an SMTP mail server to function. Set to `"false"` when an SMTP server is **not** available. |

### What This Variable Controls

When set to `"true"` (default), the following features are **enabled**:

| Feature | Description |
|---|---|
| **User Creation** | The **"Add User"** button appears in the Users section, allowing administrators to create new user accounts. New users receive their activation credentials via email. |
| **Email Log Notifications** | The **"Add Email"** button appears in the Logs section, allowing users to configure email notifications for `ERROR` and `FATAL` log events. |
| **Pending User Approvals** | The user approval workflow (PENDING → ACTIVE) is available for administrators. |

### When to Disable

Set `NEXT_PUBLIC_SMTP_ENABLED="false"` in the following scenarios:

- No SMTP mail server is available in the deployment environment.
- Email-based workflows (user creation, log notifications) should be temporarily suspended.
- Running the platform in an isolated or development environment without email infrastructure.
