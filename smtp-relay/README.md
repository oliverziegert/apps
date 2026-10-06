# smtp-relay

Postfix (`mwader/postfix-relay`) relaying mail from internal clients and signing
it with OpenDKIM for `pc-ziegert.de`. Its clients are the other app containers
on the `reverse-proxy` network, which reach it as `postfix-relay:25` (or
`smtp-relay-postfix-relay-1:25`). Postfix delivers directly to each
recipient's MX, with no relayhost.

## Egress through the NetBird exit node

Mail must not leave from the home line: its address is dynamic, sits on
residential blocklists and has no reverse DNS we control. The stack therefore
runs a `netbird` sidecar, and Postfix joins its network namespace
(`network_mode: service:netbird`). The sidecar is a NetBird peer in the
`mail-relays` group, which receives the `0.0.0.0/0` exit-node route on the OVH
VPS. All of Postfix's outbound traffic leaves from `141.95.41.247`
(`smtp-out.pc-ziegert.de`, matching PTR). The TrueNAS host and every other
container keep using the home connection.

- **Capabilities**: the sidecar holds `NET_ADMIN`, `SYS_RESOURCE` and
  `/dev/net/tun`, because it creates the WireGuard interface and installs
  routes and nftables rules. This is a deliberate exception to the usual
  `cap_drop: [ALL]` and is scoped to the sidecar. Postfix shares the namespace
  but not the capabilities.
- **Hostname aliases**: Postfix has no endpoint of its own on `reverse-proxy`.
  The sidecar carries the aliases `postfix-relay` and
  `smtp-relay-postfix-relay-1`, so apps keep reaching the relay under either
  name. If an alias is missing, submission breaks silently for the apps that use
  it.
- **IPv4 only**: `inet_protocols = ipv4`, because the exit node routes IPv4
  only. Without it, Postfix could prefer an AAAA MX and leave over the home IPv6
  address.
- **No open relay**: `mynetworks` is loopback plus the `reverse-proxy` subnet.
  The relay is reachable from the NetBird overlay, and abuse from the VPS
  address would get the whole VPN's control plane blocklisted or suspended.
  Overlay sources must get `Relay access denied`.
- **Outage behaviour**: if the agent or the exit node is down, mail queues and
  is retried for Postfix's default 5 days. It never falls back to the home
  line.

## Setup key

`NB_SETUP_KEY` is the `smtp-relay` setup key, declared in the infra Ansible
repo (`netbird/tenant/setup_keys.yml`). The tenant run
(`playbooks/netbird-tenant.yml`) writes its secret to
`credentials/netbird/smtp-relay.key`. Copy it into `.env` and into the
Portainer stack environment. It is not kept in the Ansible vault.

The key is only used for the first enrollment. The agent state in
`VOLUME_NETBIRD_SRC` (mounted at `/var/lib/netbird`) keeps the enrolled
identity, so restarts reuse the same peer. If that directory is lost, the
reusable key enrolls a new peer, which joins `mail-relays` on its own.

## Checks

```bash
docker exec smtp-relay-netbird-1 netbird status          # Connected
docker exec smtp-relay-postfix-relay-1 wget -qO- -4 ifconfig.co   # 141.95.41.247
docker exec smtp-relay-postfix-relay-1 postconf relayhost smtp_helo_name \
  inet_protocols mynetworks smtp_tls_security_level
```
