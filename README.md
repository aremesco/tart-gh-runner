# tart-gh-runner

One GitHub Actions job, in one throwaway macOS VM, then the VM is destroyed.

A self-hosted macOS runner that behaves like a hosted one: every job gets a
filesystem nothing has touched, and nothing a job does survives it. Built on
[Tart](https://tart.run) and Apple's `Virtualization.framework`, so a clone
boots in about ten seconds and a job starts in about fifteen.

## Why this exists

A self-hosted runner is usually a machine with the runner installed on it. That
machine accumulates. Every job inherits whatever the last one left behind, and
the failure mode is not that jobs get slower — it is that **a broken state
becomes permanent**.

Concretely, from the incident that produced this repository. A dedicated CI user
account had node and rust managed by [mise](https://mise.jdx.dev), which
provisions `rustup` and creates a shim named `rustup` pointing at itself. At
some point the shim existed and the binary did not. `rustup` on `PATH` found the
shim; the shim ran mise; mise resolved `rustup` through `PATH` and found the
shim again.

It forked **7,990 processes deep**, stopping only at `kern.maxprocperuid`. With
the per-user process table full, `tar` failed with `EAGAIN`, so jobs died in
`Set up job` with an error about tar and no mention of rust at all. It ran that
way for a day, evicting four pull requests from a merge queue, because nothing
removed the shim — nothing removed anything. The account was persistent, so one
bad minute was permanent.

Three fixes were applied to that machine: restore the missing binary, delete the
recursive shim, and cap the account's process limit. The first two were undone
by the next job, which recreated the shim. Only the process cap survived, and it
converted a wedged machine into a merely failing one.

**The property that actually fixes it is ephemerality.** A clone cannot inherit
a state that no longer exists on disk anywhere, and `mise install` provisions
the binary in the same run that creates the shim — so the inconsistent pair is
not reachable from a clean image. That is not a patch, it is the class being
removed.

Hosted runners have this property. This is how to have it locally.

## What it does

For each job:

1. mint a registration token (valid one hour, one registration)
2. `tart clone` the golden image
3. boot headless, wait for an IP
4. register the runner `--ephemeral`, so it de-registers itself after one job
5. `run.sh`, which returns when that job is done
6. destroy the VM — from a `trap`, so it happens on every exit path including
   failure and being killed

A supervisor restarting the script is what makes the next job get a clean VM.

## Requirements

- Apple Silicon. [Tart](https://tart.run) is Fair Source licensed: royalty-free
  on a personal workstation, paid above 100 CPU cores.
- macOS permits **two** VM guests per Apple host, enforced in the kernel. One
  runner needs one.
- [`gh`](https://cli.github.com), authenticated with admin on the repository —
  minting a registration token requires it.

## Install

```sh
brew install cirruslabs/cli/tart
git clone https://github.com/aremesco/tart-gh-runner
sudo cp tart-gh-runner/bin/tart-gh-runner /usr/local/bin/
mkdir -p ~/.config/tart-gh-runner
cp tart-gh-runner/tart-gh-runner.conf.example ~/.config/tart-gh-runner/config
$EDITOR ~/.config/tart-gh-runner/config
```

Then build a golden image (below), and install the supervisor:

```sh
cp tart-gh-runner/launchd/com.tart-gh-runner.plist ~/Library/LaunchAgents/
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.tart-gh-runner.plist
```

### A LaunchAgent, not a LaunchDaemon

`gh` keeps its credential in the login keyring, which a system LaunchDaemon has
no session to unlock. Run as a daemon, `gh api` returns
`{"message":"Requires authentication"}`, that JSON is passed on as a token, and
the runner reports **"New-line characters are not allowed in header values"** —
a message about HTTP headers, three layers from the cause. Using the session
that already holds the credential also means no secret on disk and no root.

## The golden image

Start from a [cirruslabs image](https://github.com/cirruslabs/macos-image-templates).
Prefer `macos-<version>-base` unless you need full Xcode: Swift ships with
Command Line Tools, and the base image is 25 GB against 66 GB.

Boot a clone and, inside it:

```sh
curl -fsSL https://mise.run | sh
mise use -g node@<version> rust@<version>      # GLOBAL, see below
printf 'export PATH=$HOME/.local/share/mise/shims:$HOME/.local/bin:$PATH\n' > ~/.zshenv
mkdir -p ~/actions-runner && cd ~/actions-runner
curl -fsSL -o r.tar.gz https://github.com/actions/runner/releases/download/v<ver>/actions-runner-osx-arm64-<ver>.tar.gz
tar xzf r.tar.gz && rm r.tar.gz
mkdir -p ~/.ssh && echo '<your public key>' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys
```

Then `tart stop` it. **Do not run `config.sh`** — a registration baked into the
image would be reused by every clone, and the service would dispatch to VMs that
no longer exist.

Two details that are easy to get wrong and hard to diagnose:

- **`mise use -g`, not just `mise install`.** A shim only resolves inside a
  directory whose config names a version. Tools that spawn from elsewhere —
  CodeQL's TypeScript extractor, for one — fail with `Could not start Node.js`
  while node is demonstrably installed.
- **`~/.zshenv`, not `~/.zprofile`.** The runner starts over a non-interactive
  ssh command, and zsh reads only `.zshenv` for those.

## Trade-offs, stated

- **One job at a time** per supervisor instance. Run several with distinct
  labels for concurrency.
- **Boot cost** of ~15s per job, against a hosted runner's queue time.
- **The host must stay awake.** A sleeping host queues jobs rather than failing
  them, and a queued check can stall anything waiting on it. On a laptop see
  [ac-sleep-guard](https://github.com/aremesco/ac-sleep-guard); `caffeinate` and
  the apps wrapping it hold idle-sleep assertions, which a closed lid ignores.
- **Toolchain versions become a property of the image.** That is more
  reproducible than whatever the host has installed, but it is a third answer
  next to your pinned versions and the hosted image's.

## Licence

MIT.
