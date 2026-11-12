# tail-hosted-gitea

[Gitea](https://about.gitea.com/) hosted privately on a Tailscale tailnet.

Gitea runs on its own. Optionally, run [tail-hosted-opengist](https://github.com/mikynov/tail-hosted-opengist) to host gists.

## Setup

1. Create `.env`:
   ```bash
   GITEA_ROOT_URL=https://gitea.{{tailnet}}.ts.net/
   GITEA_HTTP_PORT=3000
   GITEA_SSH_PORT=2222
   ```
2. Create a Tailscale auth key and store it in `secrets/tailscale_authkey`.
   * [Definition -> Tags](https://console.tailscale.com/admin/acls/visual/tags) -> Create tag
      * Tag name: **gitea**
      * Tag owner: leave empty, see [Define tag owners](https://tailscale.com/docs/features/tags#define-tag-owners)
      * Description: **Gitea**
   * [Settings -> Keys](https://console.tailscale.com/admin/settings/keys) -> Generate auth key...
      * Description: **Gitea**
      * Reusable: **yes**
      * Expiration: **90**
      * Tags: **tag:gitea**
3. Generate Gitea secrets:
   ```bash
   ./run generate_secrets
   ```
4. Setup access in Tailscale, see [Access controls](#access-controls)
5. Start containers and expose via [Tailscale Serve](https://tailscale.com/docs/features/tailscale-serve)
   ```bash
   ./run up serve
   ```
6. Create the Gitea admin user (the first registered user becomes admin)

## Notes

- `app.ini` holds only non-default values.
- `app.ini` is copied on startup to prevent modifications from Gitea's [environment-to-ini](https://docs.gitea.com/installation/install-with-docker/#managing-deployments-with-environment-variables) feature.
- Secrets are read from files (`file:///run/secrets/...`).
- Runtime state (`tailscale-state/`, `config/`, `data/*` except `data/custom/`) is git-ignored.
- For debugging purposes, uncomment the `ports` of the Tailscale sidecar to also expose Gitea on http://localhost:3000.

## Tailscale

### Access controls

Tailscale denies all traffic that is not explicitly granted. Please be aware that **by default Tailscale has "_allow all_" rule**.

1. Allow Gitea traffic

   [Policies -> General access rules](https://console.tailscale.com/admin/acls/visual/general-access-rules) -> Add rule

   * Source: **autogroup:member, autogroup:shared**
   * Destination: **tag:gitea**
   * Port and protocol: **tcp:443, tcp:2222** (HTTPS and `GITEA_SSH_PORT`)
   * Description: **Gitea**

2. Remove the default "_allow all_" rule

   [Policies -> General access rules](https://console.tailscale.com/admin/acls/visual/general-access-rules)

   Default name is "_All users and devices_" but in general the rule with _Source_ `*`, _Destination_ `*` and _Port and protocol_ `*` -> `...` -> Delete

3. Optionally, verify in [JSON editor](https://console.tailscale.com/admin/acls/file)

   Besides other settings, there should be something like:
   ```jsonc
   "tagOwners": {
      // Gitea
      "tag:gitea": []
   },
   "grants": [
      // Gitea
      {
         "src": ["autogroup:member", "autogroup:shared"],
         "dst": ["tag:gitea"],
         "ip": ["tcp:443", "tcp:2222"]
      },
   ],
   ```
   and no grant with `"src": ["*"]`

### Autogroups

List of [Autogroups](https://tailscale.com/docs/reference/targets-and-selectors?q=autogroup#autogroups)

| Autogroup | Note |
|---|---|
| `autogroup:member` | Also include owner and admins. Not included are shared (external) users and tagged devices |
| `autogroup:shared` | External users who accepted a machine-sharing invitation. Usable as a source only, and they can reach only the machines shared with them |

### Sharing

Share the machine with external users outside the tailnet, see [Share your machines with other users](https://tailscale.com/docs/features/sharing).

1. Share the machine

   [Machines](https://console.tailscale.com/admin/machines) -> `...` -> Share -> Copy share link (and send by secure channel, e.g. by [Threema](https://threema.com/) or [Signal](https://signal.org/))

   Unused links expire after 30 days. Avoid reusable invite links.

2. Recipient accepts the invitation

   The recipient needs Tailscale with MagicDNS enabled (any Tailscale account can accept).

   Once accepted, `autogroup:shared` limits the recipient to the shared machine and the ports allowed in [Access controls](#access-controls) step 1.

   Recipient can access `https://gitea.{{tailnet}}.ts.net/`

3. Optionally, revoke

   [Machines](https://console.tailscale.com/admin/machines) -> `...` -> Share -> link or email -> `...` -> Revoke invite

#### Notes

- Sharing is not membership. A user invited to the tailnet becomes `autogroup:member` and reaches Gitea (and Opengist) without any share.
- A shared machine is quarantined, it can receive connections from the recipient but cannot start any.
- Registration in Gitea is open (`DISABLE_REGISTRATION = false`), so a recipient can create an account on their own.

## Troubleshooting

### HTTPS stalls (MTU)

**Symptom:** the Tailscale machine is reachable, DNS, SSH and TCP in general work, but HTTPS hangs after the TLS _ClientHello_ and larger responses never finish.

**Cause:** Tailscale's tunnel adds a few bytes of overhead, so the network underneath must carry at least 1360 bytes. On a network with a smaller MTU, larger packets are silently dropped, so small requests work but large ones hang.

**How this setup handles it:**

- `compose.yaml` sets the Docker network MTU to 1360, the minimum Tailscale needs.
- `./run up` checks that the host `eth0` MTU is at least 1360 and refuses to start otherwise.

Links: [Tailscale issue #4833](https://github.com/tailscale/tailscale/issues/4833#issuecomment-1491119909),
[Docker bridge driver](https://docs.docker.com/engine/network/drivers/bridge/),
[Custom MTU in Docker](https://oneuptime.com/blog/post/2026-02-08-how-to-configure-docker-containers-with-custom-mtu-settings/view)
