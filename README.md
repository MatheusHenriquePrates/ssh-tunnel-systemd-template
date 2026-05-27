# ssh-tunnel-systemd-template

A systemd unit template for **SSH tunnels that actually survive failures** — without ending up in the "ssh process exists but the forward is dead" zombie state that most tutorials produce.

## Why this exists

A naive systemd unit for an SSH tunnel looks like:

```ini
[Service]
ExecStart=/usr/bin/ssh -N -L 15432:db.internal:5432 user@bastion
Restart=always
```

It works for about a week. Then one of these happens:

1. The remote port `5432` goes down for maintenance. The SSH control connection stays up. The local forward is dead but `Restart=always` never fires because the ssh process didn't exit. **The unit is "active" but nothing flows.**
2. The peer router reboots. SSH keepalives are off by default in OpenSSH client, so the TCP session sits in `ESTABLISHED` for minutes (sometimes hours, depending on `tcp_keepalive_time`) before the kernel finally times it out.
3. WireGuard is between you and the bastion. On reboot, the systemd unit races `wg-quick` and tries to ssh through an interface that doesn't exist yet — ssh exits, systemd restarts, ssh exits again, in a tight loop for 30 seconds until backoff kicks in.

Most "SSH tunnel as a service" guides on the internet have at least one of these bugs.

## The fix

Two SSH flags and one systemd dependency:

| Setting | What it does | Without it |
|---|---|---|
| `-o ExitOnForwardFailure=yes` | If the forward fails to bind at startup (or any time after, on a re-negotiation), ssh exits non-zero | Tunnel is "up" but dead |
| `-o ServerAliveInterval=30` + `ServerAliveCountMax=3` | Sends a keepalive probe every 30s, kills the session after 3 missed (= 90s) | Hangs for `tcp_keepalive_time` (2 hours default on Linux) |
| `After=wg-quick@wgX.service` + `Requires=wg-quick@wgX.service` | systemd waits for the WireGuard interface before launching ssh | Boot-time loop until backoff |
| `-o ConnectTimeout=15` | If the initial TCP handshake takes > 15s, give up | Hangs on hairpin / down peers |
| `Restart=always` + `RestartSec=10` | Once ssh exits (for any reason), restart in 10s | One-shot failure stays failed |

Result: the tunnel **either flows or systemd is actively trying to bring it back**. There is no third state.

## The two examples

### `examples/simple-tunnel.service`

Plain SSH tunnel from this host to a remote database via a bastion. No VPN in the middle. Use this when:

- The bastion is reachable over the public internet (or a VPC you already trust)
- You just need a stable `127.0.0.1:LOCAL_PORT` proxy to something inside the remote network

```
[this host] ──ssh──► [bastion] ──tcp──► [10.0.0.10:5432]
                                          ▲
                          local app on this host hits 127.0.0.1:15432
```

### `examples/wg-then-tunnel.service`

Same SSH tunnel, but the bastion is reachable only **through WireGuard**. This is the case when:

- The peer's "public" entry point is itself a WireGuard endpoint (typical for client-site VPNs)
- The remote LAN you actually need (`192.168.1.0/24` in the example) is reachable only from the WireGuard peer

```
[this host] ──wg0──► [10.10.0.2 jumpbox] ──ssh──► [192.168.1.10:5432]
                            ▲
                  this host hits 127.0.0.1:15432
```

The critical lines are at the top of the unit:

```ini
After=network-online.target wg-quick@wg0.service
Requires=wg-quick@wg0.service
```

`Requires` makes systemd refuse to start the tunnel if `wg-quick@wg0` failed to come up. `After` makes systemd wait for `wg-quick@wg0` to **finish** activating before launching ssh. You need both.

## Install

```bash
# 1. Copy the example that matches your case
sudo cp examples/simple-tunnel.service /etc/systemd/system/my-tunnel.service

# 2. Edit the Environment= lines (user, host, ports, key path)
sudo systemctl edit --full my-tunnel.service

# 3. Make sure the ssh key exists and is owned by the User= you specified
sudo -u tunneluser ssh-keygen -t ed25519 -f /home/tunneluser/.ssh/tunnel_key -N ''

# 4. First connection: accept the host key once (so StrictHostKeyChecking=accept-new can write to known_hosts)
sudo -u tunneluser ssh -i /home/tunneluser/.ssh/tunnel_key -p 22 bastion@bastion.example.com true

# 5. Start and enable
sudo systemctl daemon-reload
sudo systemctl enable --now my-tunnel.service
sudo systemctl status my-tunnel.service
```

## Verifying the tunnel is actually alive (not just "active")

`systemctl status` only tells you the ssh process is running. To know the forward is flowing:

```bash
# Is the local port listening?
ss -tlnp | grep 15432

# Can you talk to it?
nc -zv 127.0.0.1 15432

# How many forwards is the ssh session actually proxying?
sudo ss -tnp | grep " ssh.*pid=$(pgrep -f 'ssh -N.*-L 15432') "
```

For databases, the real test is a one-shot connect:

```bash
PGPASSWORD=... psql -h 127.0.0.1 -p 15432 -U appuser -d mydb -c 'select 1'
```

## Hardening notes

The example uses `StrictHostKeyChecking=accept-new` — this writes the host's key to `~/.ssh/known_hosts` on the **first** connection only, and then enforces it strictly. Safer than `no` (TOFU instead of "never check"). If you can pre-provision `known_hosts` (e.g., via config management), use `StrictHostKeyChecking=yes` and skip the first-connect step in the install.

The SSH key should be a **dedicated key for this tunnel only**, not a general-purpose user key. On the bastion side, restrict it in `authorized_keys`:

```
no-pty,no-X11-forwarding,no-agent-forwarding,permitopen="10.0.0.10:5432" ssh-ed25519 AAAA... tunnel-key
```

`permitopen` makes the key useless for anything but forwarding to that one address — so even if the key file leaks, the blast radius is one port on one machine.

## License

MIT. See [LICENSE](LICENSE).
