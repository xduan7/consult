# consult

Run an AI agent CLI (`codex` or `claude`) with your **project files read-only**,
while keeping the network and all the agent's normal abilities on. A thin guard
for delegating work to an agent without letting it change your files.

Enforcement is at the OS level via [`nono`](https://nono.sh) — not by limiting
the agent's tools or trusting it to behave.

## Install

Requires [`nono`](https://nono.sh) on your `PATH`.

```sh
ln -sf "$PWD/consult" ~/.local/bin/consult   # or anywhere on your PATH
```

The wrapper finds its `profiles/` by absolute path next to itself (symlinks are
resolved), so it works from any directory.

## Usage

The **first word is the agent**. After that, the rule is: **dash means raw,
word means prompt**.

```sh
# Simple mode — identical shape and behavior for both agents.
# The prompt is a literal string OR a path to a file holding it.
consult codex  "review the auth flow"        # answer -> terminal
consult claude "review the auth flow"        # identical
consult codex  ./prompt.md /tmp/answer.txt   # prompt from file, answer -> file
consult claude ./prompt.md /tmp/answer.txt   # identical

# Raw mode — a leading '-' passes everything through to the agent untouched.
consult codex  --help                        # codex's own help
consult claude --model opus -p "..."
consult codex  -- exec --json "..."          # '--' for bare-word subcommands

# No arguments after the agent: interactive session, sandboxed.
consult claude

consult --help                               # this tool's help
```

In simple mode the plumbing is applied for you (`codex exec
--skip-git-repo-check` reading stdin / `claude -p --strict-mcp-config`), so
both agents take the exact same command shape: prompt in, answer out. The
optional output file receives the agent's **final answer** and must live
*outside* the protected directory (it's read-only — consult refuses
otherwise). Note codex won't insist on a git repo in simple mode.

## What it guarantees

- ✅ The **directory you run it in cannot be modified** by the agent. Run
  `consult` *from the project you want protected* — not from `$HOME` or a parent
  of a writable allowed path (`~/.codex`, `~/.claude`, caches), or those
  subpaths stay writable.
- ✅ **Internet + all the agent's abilities** stay on.
- ✅ **Raw mode forwards your flags verbatim**; simple mode adds only the
  plumbing shown above, nothing else.
- ✅ **Fail-closed**: if `nono` is missing, it refuses rather than run the agent
  unsandboxed. Before each run it **self-tests** the sandbox (a throwaway write
  probe in the current directory) and refuses — telling you *why* — if the
  sandbox didn't start or the directory came back writable (e.g. a project that
  lives inside `/tmp` or a cache dir).

## What it deliberately does NOT do

- It does not lock the agent's own settings folder (`~/.codex` / `~/.claude`),
  and it does not block the agent from sending data out. **Scratch space stays
  writable** too — `/tmp`, caches, and anything you point the agent at with a
  flag like `codex -C <dir>`. That is the same exposure as running the agent
  normally — `consult` is an added guard on your project files, not a
  replacement for trusting the agent. Behavior is identical for both agents.

## Scope

macOS. Linux uses the same mechanism (nono supports it) but the profiles need
Linux group variants and testing before it's supported.

## How it works

`consult codex "review the auth flow"` runs (roughly):

```sh
nono run -s --profile profiles/consult-codex.json --allow-cwd \
  -- codex -c sandbox_mode="danger-full-access" exec --skip-git-repo-check -
# with the prompt fed on stdin; claude expands to: claude -p --strict-mcp-config
```

The profile (`profiles/consult-<agent>.json`) is the permission list nono
enforces: project dir read-only (`workdir: read`), the agent's own config dir
read+write, the system/runtime paths it needs to start readable, network on.

For **codex** specifically, consult injects `-c sandbox_mode="danger-full-access"`.
Codex normally wraps every command it runs in its own macOS sandbox, but that
can't nest inside nono's sandbox (macOS forbids a sandbox creating a sandbox), so
codex would fail to run any command. This setting tells codex not to build its own
sandbox — nono remains the single, enforcing sandbox. It is **not** a loosening:
codex's write attempts are still denied by nono (verified). Claude needs no such
injection.

One consequence: don't pass your own codex sandbox flags (`-s`/`--sandbox` or
`-c sandbox_mode=...`). They come after the injected setting and override it, so
codex tries to build its own sandbox again and every command it runs fails with
`Operation not permitted`. Your files stay protected either way — nono doesn't
care what codex thinks its sandbox mode is.
