# ghostrunner

Ephemeral GitHub Actions runners on demand. Here when you need them, gone when you don't.

You have a workstation. It's faster than the box you normally host CI runners on. When you're actively working, you want *this* box picking up the jobs so you stop waiting on queue. When you're done, you want it gone, no stale registration hanging around.

```
ghostrunner up           # register + start this box as a runner for the current repo
ghostrunner down         # deregister + kill
ghostrunner status       # running? busy? which job?
ghostrunner logs         # tail the runner's log
```

That's the whole thing.

## Why this exists

The normal self-hosted runner story is: install, register once, run as a systemd service, forget about it. That's fine for a CI server — bad for a dev workstation you don't want permanently claimed by GitHub.

`ghostrunner` inverts it:
- **No permanent registration.** The runner lives for the duration of your working session.
- **Per-repo scoping.** You can `up` for multiple repos in parallel; they don't collide.
- **One command.** No `config.sh --unattended --url ... --token $(curl ...)` dance.
- **Clean teardown.** `down` deregisters via the API and removes all local state.
- **No MCP/agent plumbing.** It's a single bash script. Claude invokes it via Bash.

## Install

```bash
git clone https://github.com/deucebucket/ghostrunner
sudo ln -s "$PWD/ghostrunner/ghostrunner" /usr/local/bin/ghostrunner
```

Or without symlink:
```bash
curl -L https://raw.githubusercontent.com/deucebucket/ghostrunner/main/ghostrunner -o ~/.local/bin/ghostrunner
chmod +x ~/.local/bin/ghostrunner
```

## Usage

### First run

```bash
cd my-project          # any repo with GitHub origin
ghostrunner up         # infers owner/repo from git remote
```

Output:
```
ghostrunner: downloading actions-runner v2.328.0
ghostrunner: registering ghostrunner-yourbox-1234 at deucebucket/my-project
ghostrunner: starting runner
ghostrunner: up. pid=12345 name=ghostrunner-yourbox-1234
ghostrunner: labels: ghostrunner,ghostrunner-yourbox,self-hosted,linux,x64
ghostrunner: target workflows with: runs-on: [self-hosted, ghostrunner]
```

### Route workflows to this box

In any workflow you want this runner to pick up, use a label that your ghostrunner has. The default labels include `self-hosted`, `linux`, `x64`, `ghostrunner`, and `ghostrunner-<hostname>`:

```yaml
jobs:
  test:
    runs-on: [self-hosted, ghostrunner]    # any ghostrunner, any host
    # or target a specific box:
    # runs-on: [self-hosted, ghostrunner-myworkstation]
    steps:
      - uses: actions/checkout@v4
      - run: npm test
```

### Tear down

```bash
ghostrunner down
```

Stops the runner, deregisters it from the repo, wipes local state. Nothing left behind.

### Check status

```bash
ghostrunner status
```

```
deucebucket/my-project: running pid=12345 since=2026-04-17T06:18:00-05:00
deucebucket/other-repo: running pid=12346 since=2026-04-17T05:01:00-05:00
```

## Token

By default `ghostrunner` uses your existing `gh auth token`. No separate PAT setup.

If you want a narrower token (e.g. a fine-grained PAT with only `actions:write` + `administration:write`), set it in the environment:

```bash
export GHOSTRUNNER_TOKEN=github_pat_...
ghostrunner up
```

## Multiple repos at once

Run `ghostrunner up` in one repo, then cd into another and run it again. Each gets its own work dir + PID file at `~/.config/ghostrunner/<owner>--<repo>/`. `ghostrunner status` lists them all. `ghostrunner down` only affects the repo you're cwd'd into (or pass explicitly).

## Safety

- Runner state lives under `~/.config/ghostrunner/` — wipe that directory if you want a total reset
- The runner binary itself is cached under `~/.cache/ghostrunner/` — re-used across `up` calls to avoid re-downloading
- `down` is idempotent: calling it on a repo that isn't registered is a no-op
- If the runner crashes, `ghostrunner status` shows `dead` for that entry; re-run `up` and it auto-cleans the stale registration

## What this doesn't do

- Doesn't auto-start on boot (that's what regular systemd runners are for)
- Doesn't handle authentication to registries — your workflows do that themselves
- Doesn't isolate the runner's filesystem from your host — it's a bare-metal runner, not a container. Don't run untrusted workflows through it.

## License

MIT — see LICENSE.
