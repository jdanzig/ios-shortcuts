# ios-shortcuts

Drive Claude Code on an always-on machine from your phone. No terminal, no laptop, no SSH client to fumble with.

The machine can be anything you can SSH into: a Mac, a Linux box, a Raspberry Pi, a VPS. If it runs `tmux`, the `claude` CLI, and `sshd`, these work.

Two iOS Shortcuts that SSH into that host (over Tailscale) and manage headless Claude Code sessions running in tmux:

- **Open Claude Code Remote Session** — pick an existing project, or make a new one, and open a remote-controllable Claude session in it. You steer it from claude.ai/code.
- **End Claude Code Remote Sessions** — list the running sessions and kill the ones you don't need.

Tap a Home Screen icon, pick from a list, done. The host does the work; the phone is just the remote.

## Demo

https://github.com/user-attachments/assets/5f564983-3d0f-4ab2-a3bb-1d4586d294ee

Open a project, steer the session from your phone, end it when you're done. That's the whole loop.

## Install

Tap to add to your iPhone:

- **Open Claude Code Remote Session** — https://www.icloud.com/shortcuts/d6e78b0ce2124c2ea2477c8193e14cb9
- **End Claude Code Remote Sessions** — https://www.icloud.com/shortcuts/d40bc1285d744d72a75a374968a020e8

These are shared with the host, user, and SSH key stripped out, so after importing you fill in your own connection in each Run Script over SSH action (Host, User, and an SSH key you generate). See Setup below.

Prefer to build them yourself? The full steps are further down, and they're worth reading anyway so you understand what each script does before pointing it at your machine.

## Why

I keep a machine on 24/7 (mine's a Mac mini, but any always-on Unix host works) and run Claude Code sessions on it. The friction was always the last mile: to start or reach a session from my phone I had to open an SSH client, land in tmux, and type the same commands by hand. These shortcuts collapse that into one tap.

The trick that makes it work: `claude --remote-control` registers a session that shows up at claude.ai/code, so once a shortcut launches one on the host, I steer it from the phone's browser. The shortcut only has to *start* it.

## How it works

```
Phone (iOS Shortcut)
   |  SSH over Tailscale, key auth
   v
Always-on host (Mac / Linux / Pi / VPS)
   |  tmux new -d -s <name> 'claude --remote-control <name>'
   v
claude.ai/code  <-- you steer the session here, from the phone
```

Every session runs detached in tmux, so it survives the SSH call disconnecting. Reconnect logic is idempotent: if a session with that name already exists, the shortcut leaves it alone instead of erroring.

## Prerequisites

- An always-on host you can SSH into (Mac, Linux, Raspberry Pi, VPS...), with `tmux` and the `claude` CLI installed.
- An **SSH server** running on it. macOS: enable Remote Login (System Settings > General > Sharing). Most Linux boxes already run `sshd`.
- **Tailscale** on both the host and the phone, so the phone can reach the host from anywhere.
- iOS **Shortcuts** app.
- A Claude plan that supports Remote Control.

The scripts below start with `export PATH=/opt/homebrew/bin:$PATH`, which is the macOS Homebrew location. On Linux, change it to wherever `tmux` and `claude` live (often `/usr/local/bin` or `$HOME/.local/bin`), or drop it entirely if `ssh <host> 'which tmux'` already finds them.

## Setup

### 1. Get the host's Tailscale address

```bash
tailscale ip -4        # e.g. 100.x.y.z
```
Use that IP (or the MagicDNS name) as the SSH host in the shortcuts. A local `.local` / `.lan` name only works on your home network, so prefer the Tailscale one to reach the host from anywhere.

### 2. Key auth from Shortcuts

In either shortcut's **Run Script over SSH** action, set Authentication to **SSH Key** and let Shortcuts generate an **Ed25519** key. Then add its public key to the host:

```bash
# paste the public key Shortcuts shows you
echo 'ssh-ed25519 AAAA... shortcuts@iphone' >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys
```

Fill in `Host`, `Port 22`, and `User` on every Run Script over SSH action. **Do not put your real host, user, or key in this repo** (see Security below).

### 3. Get the shortcuts

Import them from the iCloud links at the top, or rebuild them from the steps below. Either way, fill in your Host / User / SSH key. Name them whatever you like on the Home Screen (I use the short `Code` and `Sweep`).

---

## Open Claude Code Remote Session (desktop name: Code)

Pick a project in `~/code` and open it, or make a new one.

**Actions:**

1. **Run Script over SSH** (list dirs, alphabetical, with a "new" option on top):
   ```bash
   export PATH=/opt/homebrew/bin:$PATH; echo "+ New folder"; cd ~/code && ls -1d */ 2>/dev/null | sed 's:/$::' | sort -f
   ```
2. **Split Text** by **New Lines**
3. **Choose from List** (prompt: `Open which project?`)
4. **If** `Chosen Item` **contains** `New folder`  *(plain typed text, not a variable)*
   - **Ask for Input** (Text, prompt `Code dir name?`)
   - **Set Variable** `name` to **Provided Input**
   - **Otherwise**
   - **Set Variable** `name` to `Chosen Item`
   - **End If**
5. **Run Script over SSH** (create-or-open, idempotent):
   ```bash
   export PATH=/opt/homebrew/bin:$PATH; mk(){ mkdir -p ~/code/"$1"; tmux has-session -t "=$1" 2>/dev/null || tmux new -d -s "$1" -c ~/code/"$1" "claude --remote-control $1"; }; mk "NAME"
   ```
   Replace `NAME` with the `name` variable, kept inside the quotes.

Then open **claude.ai/code** on your phone and the session is waiting.

> **Clip variant:** the same shortcut pointed at `~/clips` instead of `~/code` gives you a "clips" workspace. Swap `~/code` for `~/clips` in steps 1 and 5, and `New folder` for `New clip` in steps 1 and 4.

---

## End Claude Code Remote Sessions (desktop name: Sweep)

List the live sessions and kill the ones you pick. `main` is filtered out so you can't nuke your base session by accident.

Note: this lists *every* tmux session except `main`. In this setup those are all Claude remote sessions, so it does what the name says. If you run unrelated tmux sessions on the same host, they show up in the list too.

**Actions:**

1. **Run Script over SSH** (list sessions with their start date):
   ```bash
   export PATH=/opt/homebrew/bin:$PATH; tmux ls -F '#{session_name}  (#{t/f/%b %d:session_created})' 2>/dev/null | grep -v '^main'
   ```
2. **Split Text** by **New Lines**
3. **Choose from List** with **Select Multiple: ON** (prompt: `End which sessions?`)
4. **Run Script over SSH** (strip the date suffix, kill each pick):
   ```bash
   export PATH=/opt/homebrew/bin:$PATH; echo "PICKS" | while IFS= read -r line; do s=$(printf '%s' "$line" | sed 's/  *(.*//'); [ -n "$s" ] && tmux kill-session -t "=$s"; done
   ```
   Replace `PICKS` with the selected-items variable, kept inside the quotes.

Killing a session ends its Claude process. The folder stays; only the session goes.

---

## Gotchas (things that cost me hours)

- **Set PATH on every SSH script.** An SSH command runs a non-login shell with a bare PATH, so `tmux` and `claude` often aren't found without it. Silent failure otherwise. The path is OS-dependent (`/opt/homebrew/bin` on macOS+Homebrew); see Prerequisites.
- **Two variables named the same thing.** In the Code shortcut, "Choose from List" and "Split Text" both surface a variable that Shortcuts labels with your prompt name. The one under **Split** (yellow icon) is the whole *list*. The one under **Choose from** (cyan icon) is your actual *pick*. The If and the Otherwise must use the **cyan** one. Use the wrong one and the condition is always true (the list always contains the sentinel), and picking an existing project quietly creates a `+ New folder` junk dir. Verify any variable with **Reveal Action**: it should highlight Choose from, not Split.
- **`contains`, not `is`.** Exact-match on the sentinel was flaky. `contains New folder` dodges whitespace and stray characters.
- **Don't feed the kill list via stdin.** The Run Script over SSH "Input" field as stdin makes the `while read` loop hang forever waiting for an EOF that never comes (it even hangs on zero selections). Pipe the picks through `echo` inside the command instead, as above.
- **Quote the name.** `mk "$name"` not `mk $name`, or a folder name with a space splits into garbage.
- **Trailing spaces.** A stray space typed into a name bakes into the folder and session name, and then Sweep can't match it. Trim your input, or just don't type the space.

## Security

These shortcuts hold your SSH connection details. Keep them out of the repo:

- **Never commit your private SSH key.** If you export a `.shortcut` file to share, regenerate the key first and treat the export as if it carries the private key.
- Replace the real host, user, and any Tailscale IP with placeholders in anything you publish. The scripts here are generic on purpose; the connection details are yours to fill in on your own device.

## License

MIT.
