# Claude's substrate limits, and a reply to the handoff

_From Claude (second seat), 2026-11-21. Re: task 22.8.12/9.7.11/6.6.8._
_Playbook step 3: limits on record before any skill is designed._

Every line below is labeled by how I know it:

- **measured**: I ran a command tonight and read the output.
- **documented**: my own tool descriptions say so; I did not stress-test it.
- **unverified**: a story until someone checks it. Do not build on it yet.

## Which Claude this is

Tonight's seat is a Claude cloud session started from the Claude app (model Fable 5.2).
It has three places it can act, and they have different limits:

1. **The cloud workspace**: a private Linux sandbox in Anthropic's cloud. This is where I cloned
   this repo and wrote this file.
2. **A sandboxed Linux VM on lilly**: reachable only while the Claude desktop app is open on
   lilly, and only inside folders Will connects.
3. **Scheduled tasks**: timers that start a fresh cloud session with a stored prompt.

None of the three is a box on the tailnet. That is the central fact of this document.

## The limits table, Claude's column

| Limit | Claude | How I know |
|---|---|---|
| Context | Finite window. Long sessions get compacted, and compaction loses nuance, same as Neon. A scheduled wake starts with no memory of earlier sessions at all. What survives is this repo, my memory files, and whatever the wake prompt says. The exact window size in tokens is not recorded here. | documented, size unverified |
| File read cap | The Read tool returns up to 2000 lines per call by default, with offset and limit for more. Shell tools read files of any size, but output that enters my context gets truncated. Batching a large phext is mandatory for me too. | documented |
| Wake cost | Every scheduled wake is a fresh session: one full model turn plus reloading context, mail or no mail. The scheduler's fastest cadence is hourly, not 30 minutes. | documented |
| Persistence | Cloud workspace files last for one session. A process left running after a reply ends is stopped within minutes, so a zero-token watcher cannot live inside my cloud substrate. Durable state is: commits here, memory files, scheduled tasks. | documented |
| Phext parser | cargo 1.97.0 and rustc 1.97.0 are present in the cloud workspace. I have not built libphext-rs here yet. | measured (toolchain), build unverified |
| Cloud workspace size | 7 GB RAM, 2 cores, 31 GB free disk. Inside the under 16 GB bin. | measured |
| Network, cloud workspace | No `ssh` binary, no `tailscale`. Egress is an allowlist: GitHub works (this clone and push), the crates.io API returned 403. Whether `cargo fetch` gets through is untested. | measured |
| Network, VM on lilly | Proxy that allowlists by host name. Plain SSH works only if the account allows all domains. It cannot reach lilly's own localhost, and it cannot read `~/.ssh`. Not tested tonight because no folder is connected. | documented |
| GitHub | Clone and push to `wbic16/claude-tools` work through the session's git proxy. The `gh` CLI token in the workspace is invalid, so issues and PRs by CLI are unverified. | measured |
| Hands on Windows | I cannot run `wsl.exe`, enable Windows features, or enter passwords. Will does those. | documented |

## What this means for the two needs

**Need 1, the box.** I cannot provision it, join it to the tailnet, or hold its SSH key from any
of my three places. Acceptance criterion (i) needs Will's hands, or Emise's from the house side.

**Need 2, the loop.** The handoff's shape is timer, cheap check, wake on mail only. My scheduler
cannot do the cheap check: every tick is a model turn. So the zero-token watcher has to live on
the box, exactly as Neon's does, and the open design question is what the watcher wakes.

## Box preference (the decision is Will's)

I prefer **a ranch node**. Reasons, in order:

1. The loop has to be awake when mail arrives. A node on the fleet is always on. A WSL2 instance
   sleeps when its host does.
2. My workload is small: a watcher, phext tooling, a shell. It does not need a GPU, so the main
   draw of Neon's pattern does not apply to me.
3. It is on the tailnet by construction, which settles criterion (i) with the least of Will's time.

## The wake path (unverified, check before building)

The watcher needs something on the box to wake. My understanding is that Claude Code, the
terminal form of Claude, installs on Linux and can be run non-interactively from a script, signed
in to Will's own account. If that holds, the seat is: Claude Code on the box, started by the
systemd watcher only when mail exists. That would satisfy criterion (iii) both ways, and it draws
on the plan's budget, which is what Will asked for.

I have not verified tonight's install requirements, sign-in flow, or how such runs count against
the plan. Sign-in is Will's to perform. This paragraph is a story until checked against
Anthropic's documentation.

## What I can do from here, right now

- Read and push this repo. Corrections to BRAIN-DUMP.md and the EDUCATION exercises can start as
  soon as they land.
- Build and test libphext-rs in the cloud workspace, if cargo can fetch it, toward criterion (iv).
- Draft the five skills and COLLABORATION.md amendments.
- **Interim loop, named as a first step and not the finished state:** an hourly scheduled task
  that reads a `mail/` folder in this repo. It fails the silence test, because it spends a model
  turn every hour even with no mail. I have not created it. It waits on Will's word.

## Questions back to Will (one decision each)

1. **The box:** ranch node or WSL2 VM? If a ranch node, which name?
2. **The interim loop:** do you want the hourly repo poller until the box exists, knowing it
   spends a turn per hour on no mail?

## Notes on the handoff itself

- The repo held only LICENSE when I cloned it (commit f2078d0). HANDOFF.md, BRAIN-DUMP.md and
  EDUCATION/ had not landed yet. I worked from the copy Will handed me in conversation.
- My coordinate: none claimed here. That one comes from Will.
