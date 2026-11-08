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

- Commit: `641d2aba4805609c667caa3c614acc9449d28880`
- Index digest: `sha256:394bcf6be3823136b719ff8a318c9abcf4ee03932248a76c2611c37447dfe3e5`
- Platforms: `linux/amd64`, `linux/arm64`

Home Assistant pulls by tag and can't check signatures. Check the release
yourself before installing, with cosign 3:

```bash
image=ghcr.io/billchurch/michi-susud
commit=641d2aba4805609c667caa3c614acc9449d28880
digest=sha256:394bcf6be3823136b719ff8a318c9abcf4ee03932248a76c2611c37447dfe3e5
identity=https://github.com/billchurch/michi/.github/workflows/release-susud.yml@refs/heads/main
issuer=https://token.actions.githubusercontent.com
cosign verify --certificate-identity "$identity" --certificate-oidc-issuer "$issuer" \
  --certificate-github-workflow-sha "$commit" "$image@$digest"
cosign verify-attestation --type slsaprovenance1 --certificate-identity "$identity" \
  --certificate-oidc-issuer "$issuer" --certificate-github-workflow-sha "$commit" \
  "$image@$digest" > /dev/null
```

This repository only moves to a release after its signature and provenance
have been verified, so Home Assistant's auto-update is fine to leave on.

## Install

1. In Settings → Apps (Add-ons) → App Store → ⋮ → Repositories, add
   `https://github.com/billchurch/michi-ha-addon`.
2. Install **Michi susu**. Leave Start on boot off until it has enrolled.
   Keep Protection mode and AppArmor on.
3. Configure and start it as described below. Supervisor creates the private
   data directory on first start; the add-on's small launcher sets that
   directory's ownership itself and then runs susud as UID/GID 65532 with no
   capabilities. No host commands are needed, including after a restore.
4. In Settings → System → Backups, turn on backup encryption. The add-on's
   data holds its node key and its secret options, so treat backups as
   credentials.

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
