# TLS certificates

nginx mounts this folder at `/etc/nginx/ssl` (read-only) and expects two files:

| File | Contents |
|---|---|
| `fullchain.crt` | Server certificate **+ intermediate chain** (fullchain), in PEM. |
| `private.key`   | The matching private key, in PEM. |

The files shipped in this repository are **placeholders** (example content).
Replace them with your real certificates before bringing the stack up.

Ways to obtain them:
- A certificate issued by your CA / provider (bring-your-own-cert) → place them here with these names.
- Let's Encrypt (Certbot) → copy `fullchain.pem` and `privkey.pem` to `fullchain.crt` and `private.key`.
- No certificate management: use **Caddy** as the reverse proxy (automatic HTTPS). See manual Section A7.

> Never commit the real private key to version control. The `.gitignore` already excludes `*.crt` and `*.key`.
