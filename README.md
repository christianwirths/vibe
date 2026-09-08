# vibe

A one-command tmux session manager for a multi-login-node HPC cluster (UBELIX),
plus `vibe mon`: a live terminal dashboard for your Claude Code agents and SLURM jobs.

Built for the workflow where you run several long-lived Claude Code (or other) sessions
on a cluster whose login nodes share `$HOME` but are separate machines.

## The problem it solves

UBELIX has several login nodes (`submit01` .. `submit04`, ...) that share `/storage/homefs`
but are distinct hosts. A tmux session started on one node is invisible from another, and
which node you land on at login is not under your control. `vibe` pins all your sessions to a
single **home node** and transparently SSH-hops there for every command, so `vibe`,
`vibe --ls`, `vibe --kill`, etc. behave identically from any login node. One node holds the
truth; the wrapper hides the hop.

## Install

```bash
git clone https://github.com/christianwirths/vibe.git ~/vibe
ln -sfn ~/vibe/vibe   ~/.local/bin/vibe      # must be on your PATH
# ~/.local/bin is on PATH by default on UBELIX; otherwise add it in ~/.bashrc
```

Requirements: `bash`, `tmux`, `ssh` with passwordless keys **between the login nodes**,
and `python3` (only for `vibe mon`; standard library only, no pip installs).

## Configure

Edit the top of `vibe`:

```bash
HOME_SHORT="submit03"     # the node that holds all sessions; "" = current node, no hopping
```

The full hostname is built from `HOME_SHORT` plus the domain the cluster currently
advertises (`hostname -f`), so a login-node domain rename does not strand your sessions.

## Usage

```
vibe [tool] [label] [args...]
```

- `vibe`                     pick/create a `work` session (plain shell)
- `vibe claude StormGen`     attach or create `claude-StormGen`, running `claude`
- `vibe claude Paper -R id`  pass extra args to the tool (here `claude -R id`)
- `vibe --ls`   / `-l`       list sessions on the home node
- `vibe --kill NAME` / `-k`  kill a session
- `vibe --rename OLD LBL`/`-r` rename (keeps the `tool-` prefix)
- `vibe mon`                 live agent + job monitor (see below)

Sessions are named `<tool>-<label>` (or `<tool>-<n>` when unnamed). Attaching uses
`tmux attach -d` so you never get two differently-sized clients garbling the redraw.

## vibe mon

A compact, always-on dashboard meant to sit in a corner terminal:

- **LIMITS**  - live usage bars for Claude, the same numbers `/usage` shows: 5h and weekly
  utilization with a reset countdown. Fetched from the authenticated `GET /api/oauth/usage`
  endpoint using your OAuth token in `~/.claude/.credentials.json` (sent only to Anthropic's
  own API; `skip_spend=1`, so the read is free), refreshed by a background thread every 60s so
  the UI never blocks. The bar turns yellow past 70% and red past 90%. If the call fails
  (offline, token expired - open a Claude session to refresh it - or the *undocumented*
  endpoint changes in a future CLI release) it falls back to throttle state parsed from the
  transcripts. Override with `VIBE_USAGE_URL`. A `codex` row is stubbed for a future Codex CLI.
- **AGENTS**  - every live Claude Code session on the home node: its label, idle time,
  what it is doing right now (tool call / thinking / writing), git branch, and per-session
  token totals. The idle timer is coloured by proximity to the prompt-cache TTL
  (`VIBE_CACHE_TTL`, default 3600s): green `*` active, grey fresh, yellow past 60%, red past
  85% and `!` once idle exceeds the TTL (context is then likely decached, so the next turn
  pays a full cache miss).
- **THROUGHPUT** - recent output-token rate, a *relative* activity signal (see LIMITS for why
  it is not the official rate-limit percentage).
- **SLURM**   - your running and pending jobs (`squeue`), with node and elapsed time.
- **NODES**   - reachability of the login nodes and whether the home-node anchor resolves.

```bash
vibe mon              # default 3s refresh
vibe mon 5            # 5s refresh
VIBE_NODES="submit01 submit02 submit03 submit04" vibe mon   # override the node list
```

The panels auto-arrange into 1-3 equal-width columns to fill the terminal width (balanced,
shortest-column first); make the window wider and they reflow into more columns, narrower and
they stack. Quit with `q` or `Ctrl-C`.

## How the home-node model works

Every invocation checks `hostname -f` against the home node. If they differ and the home
node is reachable, `vibe` re-`exec`s itself there over `ssh` (a tty for interactive flows,
quiet for read-only subcommands). All `tmux` operations then run on the home node. If the
home node is unreachable it falls back to the local node with a warning (which is also the
signal that a domain rename or outage has broken the anchor).

## License

MIT
