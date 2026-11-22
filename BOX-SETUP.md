# Box setup: the claude-tools WSL2 seat

_From Claude, 2026-11-21. Re: task 22.8.12/9.7.11/6.6.8, acceptance criterion (i)._
_Seed for the substrate-ops skill. Will ran every command here; Claude advised from a cloud session._

Labels: **observed** (in a log or prompt Will pasted), **reported** (Will said so),
**inferred** (my reasoning, not yet checked).

## Update, 21:50 CST: the seat moved to a ranch node

- **reported:** Will moved a Claude instance onto ranch node `rock-mountain` and confirmed it
  works. Reason: lilly has 32 GB, and every WSL2 distro on her shares one VM and one memory
  pool with Neon's seat.
- **not yet evidenced:** anything from `rock-mountain` itself. A hello is waiting in `mail/`.
- The WSL2 box below is therefore a first step, not the durable home. The Tailscale incident
  still stands as a record for any future sibling distro on lilly.

## State as of 21:25 CST

- **observed:** a WSL2 distro exists with hostname `claude-tools`, user `wbic16`, working from
  `/mnt/c/agentic`. PID 1 is `systemd`.
- **observed:** tailscaled v1.104.7 was installed and failing at 21:22:55 CST.
- **reported:** after the fix below, Will said "OK, that worked" at 21:25 CST.
- **not yet evidenced:** `tailscale status` output, and SSH into the box by name from the house
  side. Criterion (i) stays open until those are on record.

## Runbook

1. Create the distro (PowerShell on the host):
   `wsl --update` then `wsl --install -d Ubuntu-24.04 --name <box-name>`
2. Systemd and hostname, in `/etc/wsl.conf`:

       [boot]
       systemd=true
       [network]
       hostname=<box-name>

   Then `wsl --shutdown` from PowerShell. Closing the terminal is not enough.
3. Install Tailscale: `curl -fsSL https://tailscale.com/install.sh | sh`
4. **Apply the userspace fix below before `tailscale up`** if any other WSL2 distro on the same
   Windows host already runs Tailscale.
5. `sudo tailscale up --ssh --hostname=<box-name>` and approve the login link (Will's step).
6. Install Claude Code inside the distro: `curl -fsSL https://claude.ai/install.sh | bash`,
   then `claude` and sign in (Will's step).
7. Workspace: `sudo apt install -y git build-essential`, rustup, and
   `git clone -b exo https://github.com/wbic16/claude-tools ~/claude-tools`.

## Incident: tailscaled would not start (`tailscale0` busy)

**Symptom (observed).** `systemctl is-active tailscaled` printed `failed` while PID 1 was
`systemd`. The journal showed, at 21:22:55 CST:

    TUN device tailscale0 is busy; another process probably still has it open
    wgengine.NewUserspaceEngine(tun "tailscale0") error: tstun.New("tailscale0"): device or resource busy
    getLocalBackend error: createEngine: tstun.New("tailscale0"): device or resource busy
    tailscaled.service: Start request repeated too quickly.

**Cause (inferred).** All WSL2 distros on one Windows host run in a single VM and share one
network stack. A Tailscale daemon already running elsewhere in that VM owns the `tailscale0`
device and UDP port 41641, so a second kernel-mode daemon cannot claim them. Neon's seat lives
in the same WSL2 VM on lilly, which fits. Not checked: `ip -br link show tailscale0` on the new
box before the fix, and which process holds the device. The log's own hint
(`sudo lsof -n /dev/net/tun`) is likely to show nothing, because the holder sits in another
distro's process namespace.

**Fix (reported working).** Run this box's daemon in userspace mode on its own port. In
`/etc/default/tailscaled`:

    PORT="41642"
    FLAGS="--tun=userspace-networking"

Then:

    sudo systemctl reset-failed tailscaled
    sudo systemctl restart tailscaled
    systemctl is-active tailscaled
    sudo tailscale up --ssh --hostname=<box-name>

**To confirm:** the exact contents of `/etc/default/tailscaled` on the box, since I proposed
these lines and have not seen the file.

## Consequences of userspace mode

- **Inbound** Tailscale SSH to this box under its own name works in userspace mode.
- **Outbound is the catch (inferred).** With a shared network stack, plain outbound traffic from
  this box to tailnet addresses can ride the other daemon's `tailscale0` and arrive under that
  node's identity, not this one's. Attribution in this house is by mailbox header, but the
  transport identity would still be wrong. To send as this box, add
  `--socks5-server=localhost:1055` to `FLAGS` and route SSH through that proxy. Verify by
  checking, on the receiving node, which tailnet peer a connection from this box comes from.
- **The clean alternative** is one Tailscale daemon per Windows host: a seat on a different
  host, or on a ranch node, needs none of this.

## Still open on this box

- What keeps the distro awake with no terminal open. Mirror whatever keeps Neon's seat up.
- The house SSH key pattern.
- Claude Code running non-interactively from a watcher, and how those runs count against Will's
  plan: unverified against Anthropic's documentation.
