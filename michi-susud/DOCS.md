# Michi susu

A Home Assistant add-on that runs **susud**, michi's vantage-point node
(a *susu*). It dials out to a michi broker over WSS and runs read-only
network checks (DNS, ping, TCP, HTTP headers, TLS, traceroute, mtr, local
network state) when the household's michi controller asks. It listens on
nothing and needs no port forwarding. It runs on `aarch64` and `amd64`.

It is only useful with a michi controller and broker you run yourself.

## The image

The add-on pulls `ghcr.io/billchurch/michi-susud:<version>`, where
`version` is a full commit SHA from a signed release:

- Commit: `0cf918eba47cbe4d74fea0a16f837be9c1012566`
- Index digest: `sha256:f8f0565900c01c3522dc1e9b197b7588637c639e1a23bf43940c804a45cdd681`
- Platforms: `linux/amd64`, `linux/arm64`

Home Assistant pulls by tag and can't check signatures. Check the release
yourself before installing, with cosign 3:

```bash
image=ghcr.io/billchurch/michi-susud
commit=0cf918eba47cbe4d74fea0a16f837be9c1012566
digest=sha256:f8f0565900c01c3522dc1e9b197b7588637c639e1a23bf43940c804a45cdd681
identity=https://github.com/billchurch/michi/.github/workflows/release-susud.yml@refs/heads/main
issuer=https://token.actions.githubusercontent.com
cosign verify --certificate-identity "$identity" --certificate-oidc-issuer "$issuer" \
  --certificate-github-workflow-sha "$commit" "$image@$digest"
cosign verify-attestation --type slsaprovenance1 --certificate-identity "$identity" \
  --certificate-oidc-issuer "$issuer" --certificate-github-workflow-sha "$commit" \
  "$image@$digest" > /dev/null
```

Keep automatic updates off and only move to another verified release SHA.

## Install

1. In Settings → Add-ons → Add-on Store → ⋮ → Repositories, add
   `https://github.com/billchurch/michi-ha-addon`.
2. Install **Michi susu**. Leave Start on boot off until it has enrolled.
   Keep Protection mode and AppArmor on.
3. Prepare the add-on's data directory once. Supervisor creates it owned by
   root; the image runs as UID/GID 65532. From the HAOS host (the console,
   or developer SSH on port 22222), find the add-on's slug with
   `ha addons list` and run:

   ```bash
   chown 65532:65532 /mnt/data/supervisor/addons/data/<slug>
   chmod 0755 /mnt/data/supervisor/addons/data/<slug>
   ```

   Change only this directory, never recursively. Without host access, a
   terminal add-on with Protection mode temporarily off can do the same
   through Docker:
   `docker run --rm -v /mnt/data/supervisor/addons/data:/d alpine:3.22 sh -c 'chown 65532:65532 /d/<slug> && chmod 0755 /d/<slug>'`.
   Turn Protection mode back on afterwards.
4. In Settings → System → Backups, turn on backup encryption. The add-on's
   data holds its node key and its secret options, so treat backups as
   credentials. A restore must keep the directory's ownership; repeat step 3
   if needed.

The host's `net.ipv4.ping_group_range` must include 65532 for ping.

## Configure and enrol

| Option | Value |
| --- | --- |
| `node_name` | The node name the broker expects, e.g. `susu-casita` |
| `broker_host` | Your broker's hostname only: no scheme, port or path |
| `controller_hosts` | Every name and address of your controller host (1 to 16); susud refuses to probe them. Each must resolve at startup |
| `access_client_id`, `access_client_secret` | This node's Cloudflare Access service token |
| `enrol_token` | A one-time enrolment grant from your michi console (valid for an hour); clear it after enrolment |
| `deny_extra` | Extra CIDRs susud refuses to probe. Defaults to the Tailscale ranges (`100.64.0.0/10`, `fd7a:115c:a1e0::/48`); at most 16 |
| `mtr` | `true` to offer mtr (per-hop loss); `false` to leave it out |

Start the add-on and watch its log for `enrolled`, then an authenticated
`call-home stream up`. Clear `enrol_token`, save and restart; a non-empty
enrolment option after enrolment stops startup. Turn on Start on boot.
Don't share the logs: they name your broker and controller.

A refused grant isn't retried: clear it, get a new grant and restart. If the
node key is lost, revoke the node in your console and enrol again; never
delete `/data/node.key` to repair a connection.

## Privileges

Host networking gives the checks the host's real network view. The only
extra capability is `NET_RAW`, and only the `mtr-packet` helper holds it;
susud itself runs as UID 65532 with no effective capabilities. AppArmor is
on. The add-on asks for no Home Assistant API role, Docker socket, host PID,
devices or host folders.

Source: this repository mirrors the add-on from michi's private source
repository. Issues and changes go through the maintainer.
