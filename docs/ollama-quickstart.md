# Rejudge on Ollama, step by step

*[По-русски](ollama-quickstart.ru.md)*

From nothing to a finished review. Every step is a command plus what should come back. If something
else came back, the step ends with "When it doesn't match".

Eight steps, ten minutes. Steps 1–6 were walked through in a clean sandbox: fresh clone, empty
settings directories, a separate npm prefix.

## First, how this fits together

Three things, without which the steps below read backwards.

**It installs once, globally.** `rejudge` is a command, like `git` or `curl`. It is not a project
dependency: it never lands in `package.json` or `node_modules`, and you do not install it per
repository.

**Nothing gets linked into a project.** The connection to a project is the current directory, and
that is all of it. You go into the repository and run `rejudge`. That directory decides two things:
which config is used, and which files the reviewers can read.

**The setup is also once per machine.** `rejudge setup ollama` writes two files in your home
directory — a provider for Pi, and a panel. It writes nothing into a project.

The only thing that ever lives inside a project is an optional `.rejudge/config.json`, for when that
one repository wants a *different* panel. It is a plain file, not a link, and step 8 covers it.

---

## Step 1. Node 22 or newer

```bash
node --version
```

You want `v22.19.0` or above.

If it is older:

```bash
nvm install 22
nvm use 22
```

**When it doesn't match.** Run the check again in a new terminal window: `nvm use` only lasts for
the current shell. `nvm alias default 22` makes it stick. If other projects need the older Node,
leave the default alone — step 4 installs a wrapper that finds a suitable version by itself.

---

## Step 2. Ollama, signed in

```bash
ollama --version
ollama signin        # if you have not already
```

Check that the cloud answers on your account:

```bash
ollama run glm-5.2:cloud "ping"
```

You want a sensible reply from the model. This also starts the daemon if it was not running.

**When it doesn't match.** `402 extra usage only` means the model is outside your plan — pick a
different one in step 3. An authentication error means running `ollama signin` again.

---

## Step 3. Pull the models

You need **at least four**: three reviewers and a judge. Take them from different labs — two models
from one line make the same mistakes, and a panel of those checks nothing.

The catalogue filtered to what can both reason and call tools:
<https://ollama.com/search?c=thinking&c=cloud&c=tools>

```bash
ollama pull deepseek-v4-flash:0731-cloud
ollama pull minimax-m3:cloud
ollama pull gemma4:31b-cloud
ollama pull glm-5.2:cloud
```

**The naming rule.** A model with no tag takes `<model>:cloud`. A tagged one takes
`<model>:<tag>-cloud`. So `glm-5.2:cloud`, but `gpt-oss:120b-cloud`.

Check:

```bash
ollama list
```

**When it doesn't match.** `model not found` is almost always the name — `:cloud` and `-cloud`
swapped. Pulling a cloud model is not strictly required, since the daemon proxies it either way, but
the stub is a few hundred bytes and it puts the model in your local list, which is where step 5
looks.

---

## Step 4. Make `rejudge` a global command

Clone it somewhere permanent. This is a checkout of the tool, not part of the project you mean to
review — `~/work` will do:

```bash
git clone -b mine https://github.com/pksmall/rejudge.git ~/work/rejudge
cd ~/work/rejudge
just setup
npm link
```

You need `bun` and `just` — `brew install bun just`.

`npm link` **with no arguments** means "make this checkout the global `rejudge` command". Do not
confuse it with `npm link <name>`, which is run *inside* a project and does the opposite; you do not
need that one. No project takes part in this step at all.

Check it from anywhere except the checkout:

```bash
cd ~ && rejudge --help | head -3
```

You want the line `usage: rejudge`, and `setup ollama` further down the output. If the command is
found from your home directory, it is global, and it will be found in your projects too.

**When it doesn't match.** `rejudge: command not found` means the global npm `bin` directory is not
on your `PATH`. `npm prefix -g` prints the prefix; add `/bin` to it.

`webidl.util.markAsUncloneable is not a function` means an older Node ran. Go back to step 1, or
drop in a wrapper that picks the interpreter itself:

```bash
mkdir -p ~/.local/bin
cat > ~/.local/bin/rejudge <<'EOF'
#!/bin/sh
for dir in $(ls -d "$HOME"/.nvm/versions/node/v2[2-9]* "$HOME"/.nvm/versions/node/v[3-9]* 2>/dev/null | sort -Vr); do
    if [ -f "$dir/lib/node_modules/rejudge/bin/rejudge.js" ]; then
        exec "$dir/bin/node" "$dir/lib/node_modules/rejudge/bin/rejudge.js" "$@"
    fi
done
echo "rejudge: no Node >= 22 with the rejudge package installed" >&2
exit 1
EOF
chmod +x ~/.local/bin/rejudge
```

`~/.local/bin` has to come before the Node directory on your `PATH`.

You can install from npm instead — `npm install -g rejudge` — but then there is no `setup` command,
which only exists in this fork, and you do step 5 by hand from [ollama.md](ollama.md).

---

## Step 5. One setup command, once per machine

Run it from anywhere; it only writes to your home directory:

```bash
rejudge setup ollama --dry-run     # see it without writing anything
rejudge setup ollama               # write
```

Roughly this comes back. The numbers will be yours — they only reflect your own model list:

```
Ollama setup — 4 models declared, 0 skipped

provider  wrote /Users/you/.pi/agent/models.json
config    wrote /Users/you/.config/rejudge/config.json

panel
  reviewer  ollama/minimax-m3:cloud@high
  reviewer  ollama/gemma4:31b-cloud@high
  reviewer  ollama/glm-5.2:cloud@high
  judge     ollama/deepseek-v4-flash:0731-cloud@high
  every slot comes from a different lab
```

The command does not reach for the cloud and pulls nothing: it declares what already sits on your
machine. The model list is yours, and keeping it current is yours too.

**When it doesn't match.**

- `a panel needs 4 eligible models … found 2` — back to step 3, there are not enough.
- Bigger numbers than the example, and a `skipped` block — expected, if you have models beyond the
  four from step 3. The command declares everything usable and lists the rest with a reason, usually
  no `thinking` or no `tools`. That is not a fault.
- `some slots reuse the same lab` — the panel came together, but out of one line. It will run; it
  will not check anything. Pull a model from another vendor.
- A `local models` block — local weights made it into the panel. Read the context section in
  [ollama.md](ollama.md#purely-local-models): the daemon does not serve the window the model
  advertises, and an overflow is truncated silently.
- `cannot reach the Ollama daemon` — the daemon is not up; back to step 2.

---

## Step 6. The first real run

Something trivial first, from any directory:

```bash
rejudge "Reply with the single word READY and nothing else."
```

Progress goes to stderr with the model names, the answer to stdout. It takes 20–40 seconds. The last
stderr line gives you a `run id` for continuing the conversation.

Now a real task. **This is the whole of "connecting" Rejudge to a project** — you walk into it.
Nothing was installed into the project and nothing will be:

```bash
cd ~/your/project
git diff | rejudge "review this change"
```

Run it from the root of the repository you want reviewed: the reviewers' tools work in the current
directory, and from a neighbouring one they will not see your code.

**When it doesn't match.**

- `no config found` — step 5 did not write a config. Check you did not run it with `--dry-run`.
- `410 … was retired` — the model is gone from the service. `ollama list` still shows it, but the
  service no longer serves it: pull a replacement (step 3) and repeat step 5.
- `invalid reasoning value` — the config carries a level Ollama does not know. It takes `low`,
  `medium`, `high` and `xhigh`; it rejects `minimal`.
- The review arrives but ignores the task — the model has no `compat.supportsDeveloperRole: false`
  in `models.json`. Re-run step 5; it sets that itself.
- The run sits on one model for minutes — that happens: the panel waits for its slowest member.
  `nemotron-3-super` took 13 minutes against 2 for the others in one measurement.

---

## Step 7. Teach an agent to call Rejudge

So far Rejudge is your own command in a terminal. Getting a coding agent to use it depends on the
agent, and the difference is exactly one thing: **Pi has a native tool, everything else gets the
skills plus the same CLI.**

### Pi — the native tool and the workflows

```bash
pi install "$(npm root -g)/rejudge"
```

Pi records the path instead of copying, so the command, the `rejudge` tool and both workflows
(`/rejudge`, `/rejudge-diff`) come from one installation. Restart Pi if it was already open. Inside
Pi the agent watches progress per stage and gets more control over a run than the CLI offers.

### Claude Code — skills

Skills live in `~/.claude/skills/<name>/SKILL.md`. Symlink them to the checkout so they follow it:

```bash
cd ~/work/rejudge
mkdir -p ~/.claude/skills
ln -s "$PWD/docs/skills/rejudge" ~/.claude/skills/rejudge
ln -s "$PWD/docs/skills/rejudge-diff" ~/.claude/skills/rejudge-diff
```

That gives you `/rejudge` and `/rejudge-diff`. The first sends any question to the panel; the second
is a code review of the current diff, reported in P0–P3 buckets.

### Everything else — OpenCode, Codex, whatever you run

Nothing but Pi has a native tool, so there are two ways.

If the agent reads Agent Skills, install them through the Skills CLI:

```bash
npx skills add syabro/rejudge -g -y
```

That is a separate copy from upstream, so updating it is on you: `npx skills update -g -y`. Note
that upstream has no `setup` command — the skill does not need one, it calls `rejudge` to review,
and your setup was done in step 5.

If the agent knows nothing about skills but can run commands, that is enough. Tell it to run
`rejudge` from the project root and read the answer off stdout; the file
[`docs/skills/rejudge/SKILL.md`](skills/rejudge/SKILL.md) works as the instruction — it covers how
to call it, what not to do, and how to present the result. Codex gets its own paragraph in there
about running in a TTY.

**What matters for any agent.** A run takes minutes, so give it a generous timeout and do not send
it to the background. Reviewers are read-only; `--unsafe` / `--full` hand them write access and
bash, and there is no reason to pass either without one.

---

## Step 8. Optional

**A different panel in one project.** The only thing that ever goes inside a project, and only if
you want it: `<project>/.rejudge/config.json`. It overrides the global one silently, so when a
result surprises you, check the `config:` line in the output. Write it by hand, or from the project
root:

```bash
cd ~/your/project
rejudge setup ollama --project        # panel into this project's .rejudge/config.json
```

This is still not installing Rejudge into a project — only a settings file.

**Reasoning logs.** `"debugLog": true` in the config makes every run write `.rejudge/logs/*.jsonl`.
Keep that out of git.

**Refreshing the models.** After every `ollama pull` or `ollama rm`, run `rejudge setup ollama`
again — it rewrites the provider for the new list. The config is left alone; replacing the panel
needs `--force`.

---

## Worth keeping in mind

The global command is a symlink to this checkout, because you installed it with `npm link`. So it
follows whatever branch is checked out: switch the checkout to `upstream/main` and `setup ollama`
disappears, since upstream does not have it. After editing `src/`, rebuild with `just build`.

No keys anywhere: cloud models go through the local daemon, and the daemon holds the credentials.

The reasoning behind each setting, the naming rule, a failure table and the local-context trap are
all in [ollama.md](ollama.md).
