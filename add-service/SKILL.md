---
name: add-service
description: >-
  Checklist for adding a new Homelab service. Ensures all necessary configuration
  file changes are made without omission when adding or removing a service.
  Triggers: 'add service', 'new service', '서비스 추가' (add service), '서비스 제거' (remove service),
  'remove service', 'add to docker-compose', 'add to HAProxy', 'configure Tailscale serve'.
---

# Service Add/Remove Checklist

## When Adding (in order)

1. **docker-compose.yml** — Add service definition (network: `edge` is required)
2. **haproxy.cfg** — In fe_http, add short hostname ACL + redirect. In fe_https, add FQDN ACL + `use_backend`, and add backend block.
3. **scripts/renew-ts-cert.sh** — Add short hostname to the `DOMAINS` array.
4. **Tailscale serve** — Requires sudo, only present the command to the user.
   ```
   sudo tailscale serve --service=svc:<name> --tcp=443 tcp://127.0.0.1:443
   sudo tailscale serve --service=svc:<name> --tcp=80 tcp://127.0.0.1:80
   ```
5. **Certificate Issuance** — Requires sudo, `tailscale serve` must be configured first.
   ```
   sudo tailscale cert --cert-file certs/raw/<fqdn>.crt --key-file certs/raw/<fqdn>.key <fqdn>
   ```
6. **Build PEM** — `scripts/build-haproxy-pem.sh <domain>`
7. **Update AGENTS.md** — Routing & domains list, Docker compose services list.

## When Removing

Clean up the above items in reverse order. Specifically:
- Delete the corresponding domain files from `certs/raw/` and `certs/pem/`.
- Remove from the `DOMAINS` array in `scripts/renew-ts-cert.sh`.
- Disable Tailscale serve: `sudo tailscale serve --service=svc:<name> --tcp=443 off`

## HAProxy Pattern Reference

fe_http (short hostname redirect):
```
acl host_<short> hdr(host),field(1,:),lower -i <short>
http-request redirect location https://<fqdn> code 301 if host_<short>
```

fe_https (FQDN + short hostname routing):
```
acl host_<name> hdr(host),field(1,:),lower -i <fqdn> <short>
use_backend be_<name> if host_<name>
```

backend:
```
backend be_<name>
  server <name> <container>:<port> check resolvers docker init-addr last,libc,none
```

## Important Notes

- Only suggest sudo commands, do not execute them directly (project sudo policy).
- Always check the logs after restarting a Docker service.
- Commit after completing work (Conventional Commits).
