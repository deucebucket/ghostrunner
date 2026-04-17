# ghostrunner tutorial

End-to-end walkthrough: go from zero to "my workstation just picked up the CI job" in about 3 minutes.

## 0 · Before you start

You'll need:
- `bash`, `curl`, `python3`, `tar` — on most Linux distros these are already there
- `gh` CLI, logged in (`gh auth login`) — or a PAT you set as `GHOSTRUNNER_TOKEN`
- A GitHub repo where you have `admin` access (needed to register runners)

## 1 · Install

Clone and symlink:

```bash
git clone https://github.com/deucebucket/ghostrunner ~/tools/ghostrunner
sudo ln -s ~/tools/ghostrunner/ghostrunner /usr/local/bin/ghostrunner
```

Verify:

```bash
ghostrunner version
# ghostrunner 0.1.0
```

## 2 · Register + start

`cd` into any repo with a GitHub origin and run:

```bash
cd ~/code/my-project
ghostrunner up
```

First run downloads the GitHub Actions runner binary (~100MB, one-time, cached under `~/.cache/ghostrunner/`). Subsequent runs on other repos skip the download.

You'll see:

```
ghostrunner: downloading actions-runner v2.328.0
ghostrunner: registering ghostrunner-daisy-12345 at deucebucket/my-project
ghostrunner: starting runner
ghostrunner: up. pid=12345 name=ghostrunner-daisy-12345
ghostrunner: labels: ghostrunner,ghostrunner-daisy,self-hosted,linux,x64
ghostrunner: target workflows with: runs-on: [self-hosted, ghostrunner]
```

Done. Your box is now a runner.

## 3 · Route a workflow through it

Open `.github/workflows/whatever.yml` and change `runs-on`:

```diff
 jobs:
   build:
-    runs-on: ubuntu-latest
+    runs-on: [self-hosted, ghostrunner]
     steps:
       - uses: actions/checkout@v4
       - run: ./build.sh
```

Push the change. The next workflow run will claim your box. Watch it in real time:

```bash
ghostrunner logs
```

Or target a specific host by name (useful if you have multiple ghostrunners up across different machines):

```yaml
runs-on: [self-hosted, ghostrunner-daisy]
```

## 4 · Run the same setup across multiple repos

ghostrunner is per-repo. You can be registered for several at once without collisions:

```bash
cd ~/code/frontend && ghostrunner up
cd ~/code/backend && ghostrunner up
cd ~/code/docs && ghostrunner up
```

Check what's live:

```bash
ghostrunner status
```

```
deucebucket/frontend: running pid=12345 since=2026-04-17T06:18:00-05:00
deucebucket/backend:  running pid=12346 since=2026-04-17T06:19:12-05:00
deucebucket/docs:     running pid=12347 since=2026-04-17T06:21:44-05:00
```

## 5 · Tear down when you're done

From within the repo's cwd:

```bash
cd ~/code/my-project
ghostrunner down
```

Or explicitly:

```bash
ghostrunner down deucebucket/my-project
```

Output:

```
ghostrunner: stopping runner (pid 12345)
ghostrunner: deregistering
ghostrunner: down.
```

This stops the runner process, deregisters it through the GitHub API, and removes the state directory. Nothing left claimed.

## 6 · Nuke everything (emergency reset)

If `ghostrunner` gets into a weird state:

```bash
pkill -f "actions-runner/run.sh"   # kill any runner processes
rm -rf ~/.config/ghostrunner/      # wipe state
# If any registrations are stuck, remove them manually in GitHub:
#   Settings → Actions → Runners → (delete the stale ones)
```

## Common gotchas

### "Runner already exists"

You registered, crashed, and the GitHub side still thinks you're registered. `ghostrunner up` auto-cleans stale registrations via a remove-token, but if that fails, delete the runner manually in the repo's **Settings → Actions → Runners** page.

### "Permission denied" when downloading the runner

The first `up` downloads ~100MB to `~/.cache/ghostrunner/`. Check you have write access there.

### Workflow keeps picking GitHub-hosted runners instead of mine

Check `runs-on` in the workflow — it's an array, ALL labels must match. A runner advertising `[self-hosted, ghostrunner, linux, x64]` matches `runs-on: [self-hosted, ghostrunner]`. It does NOT match `runs-on: ubuntu-latest` — that's a different pool entirely.

### My runner is registered but jobs don't arrive

Inspect the diagnostic log:

```bash
ghostrunner logs
```

The runner logs whether it's listening, connected, and what jobs it's receiving.

### Multiple machines, same repo, same hostname

If two boxes have the same hostname, their default labels collide. Set a different `GHOSTRUNNER_HOST` per box (future feature — for now, rename one host).

## Why not just use `config.sh` directly?

You can. ghostrunner is a 300-line bash wrapper around `config.sh` + the GitHub API. What it gives you:
- One command instead of four
- Per-repo scoping with automatic cleanup
- Auto-retry when a previous registration is stale
- Shared runner-binary cache across repos
- `status` / `logs` / `down` so session teardown isn't a guessing game

If you're a long-lived CI server: use the real `config.sh` + systemd. If you're a workstation that wants temporary runner-ness: ghostrunner.
