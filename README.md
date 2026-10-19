# VikingAI

A local, private AI assistant for your Linux terminal. One installer sets up
[Ollama](https://ollama.com) (the local model runtime), [ShellGPT](https://github.com/TheR1D/shell_gpt)
and LiteLLM in an isolated Python environment, and gives you a single command: `viking`.

Your prompts and logs stay on your machine. No OpenAI key is needed.

```text
$ journalctl -u ssh -n 50 --no-pager | viking "Anything suspicious in this log?"
```

## Contents

- [Supported systems](#supported-systems)
- [Prerequisites](#prerequisites)
- [Install](#install)
- [Choosing the system prompt](#choosing-the-system-prompt)
- [Usage](#usage)
- [Customization](#customization)
- [Verify the installation](#verify-the-installation)
- [What gets installed where](#what-gets-installed-where)
- [Troubleshooting](#troubleshooting)
- [Uninstall](#uninstall)
- [Security notes](#security-notes)
- [Testing status](#testing-status)
- [Manual install (other distributions)](#manual-install-other-distributions)

## Supported systems

| System | Status |
|---|---|
| Ubuntu, Debian, Kali Linux | Supported (apt-based) |
| Raspberry Pi OS **64-bit** | Supported, but CPU-only: expect slow answers, stay with small models |
| Debian derivatives (Linux Mint, Pop!_OS, Parrot, ...) | Should work: detected through `ID_LIKE` |
| 32-bit systems (armv6/armv7, i386, 32-bit Raspberry Pi OS) | **Not supported.** Ollama only ships 64-bit builds. The installer stops with an explanation |
| Fedora, Arch, openSUSE, macOS, Windows | Not supported by the installer, see [Manual install](#manual-install-other-distributions) |

Architectures: `x86_64` (amd64) and `aarch64` (arm64).

Python: 3.10 or newer is recommended. With 3.9, pip may pick an older ShellGPT; the installer
inspects the installed version and stops if it lacks the options VikingAI needs.

## Prerequisites

- A normal user account with `sudo` rights. Do **not** run the installer as root.
  `sudo` is requested only to install apt packages and to install/start Ollama.
- Internet access (PyPI, ollama.com, the Ollama model registry).
- Disk space: about 1 GB free in your home directory, plus about 4 GB free on `/usr` if Ollama is not
  installed yet. Larger models need several GB more.
- RAM: the default model (`qwen2.5-coder:1.5b`) is small, roughly 2 GB RAM is a comfortable minimum.
  Bigger models need much more. Rough rule of thumb for 4-bit models: about 1 GB of RAM per billion
  parameters, plus overhead.

Everything else (`curl`, `python3`, `python3-venv`, `zstd`, Ollama) is installed for you if missing.

## Install

Replace `<repo-name>` with the name of this repository.

```bash
git clone https://github.com/skuldexter-web/strip-your-shellgpt.git
cd strip-your-shellgpt
chmod +x install.sh
./install.sh
```

No git? Download the repository as a ZIP from GitHub, unzip it, `cd` into the folder and run the last two
commands.

The installer then:

1. Shows a banner and detects your OS, package manager and CPU architecture.
2. Checks free disk space and internet connectivity.
3. Installs missing system dependencies with apt (asks for `sudo`).
4. Installs Ollama with Ollama's official installer (`https://ollama.com/install.sh`) if needed. The script is
   downloaded to a temporary file and sanity-checked first, and you are asked to confirm.
5. Makes sure the Ollama service is running (starts it if not).
6. Asks which model to use (default `qwen2.5-coder:1.5b`; press Enter to accept).
7. Downloads the model with `ollama pull` (skipped if it is already there).
8. Creates a private Python virtual environment.
9. Installs ShellGPT and LiteLLM inside it, then inspects the installed ShellGPT version.
10. Prepares the ShellGPT <-> Ollama settings.
11. **Asks you to choose the system prompt** (see below) and creates the `viking` role.
12. Creates the `viking` launcher and, if needed, adds `~/.local/bin` to your PATH.
13. Tests everything, including a tiny question to the model, and prints a summary.

It is safe to run again. Existing installs are reused, the PATH entry is added only once, and nothing
unrelated is overwritten.

### Installer options

| Option | Meaning |
|---|---|
| `--model TAG` | Use this Ollama model tag, skip the question |
| `--prompt-mode custom\|default` | Pick the prompt option without the menu |
| `--prompt-file FILE` | Use the contents of `FILE` as your custom prompt (implies `custom`) |
| `--keep-prompt` | Leave the existing Viking role untouched (handy when only changing the model) |
| `--non-interactive`, `-y` | Never ask questions |
| `--upgrade` | Upgrade ShellGPT and LiteLLM inside the VikingAI environment |
| `--skip-test` | Skip the final model test (useful on very slow hardware) |
| `--no-modify-path` | Do not edit `~/.bashrc` / `~/.zshrc` |
| `--uninstall` | Remove VikingAI (see [Uninstall](#uninstall)) |
| `--help` | Show all options |

Environment variables: `VIKING_ALLOW_ROOT=1` (allow running as root), `VIKING_STDIN_INPUT=1` (read menu
answers from piped stdin, for automation), `NO_COLOR=1`.

## Choosing the system prompt

During installation you must choose exactly one of two options. They are mutually exclusive: the
installer never combines them.

```text
Choose your Viking system prompt:

  1) Enter my own custom prompt
  2) Use the default Viking prompt
```

**Option 1, custom prompt.** The installer pauses and asks you to type or paste your prompt. It can span
many lines. Finish by typing `END` on a line by itself (Ctrl-D also finishes).

- Your text is saved exactly as entered, line breaks included. Only the newline that comes right before
  `END` is dropped.
- Nothing from the default prompt is added, before or after.
- An empty prompt is refused. You can enter one again or explicitly switch to the default option.
- Limit: 32,768 characters.
- If you need a line that is literally `END` inside your prompt, use `--prompt-file` instead.

**Option 2, default prompt.** The built-in prompt is loaded as it is, with nothing added. It makes Viking
a senior IT engineer and cybersecurity assistant: practical help with Linux, Python, Bash, networking,
sysadmin work and programming; analysis of code, logs and error output; support for authorized pentesting,
defensive security, auditing and labs; destructive commands explained before they are suggested;
assumptions and uncertainty stated; and never claiming to have run anything.

Its last line, `APPLY MARKDOWN formatting when possible.`, is a ShellGPT convention that turns on
Markdown rendering in the terminal. If you write a custom prompt and want rendered Markdown, include that
sentence yourself.

After the installer saves the role, it re-reads the file and checks that it contains exactly the selected
prompt. The final summary states which option was used: **Custom prompt** or **Default prompt**.

Running the installer again offers the same two choices. The previous role is replaced, nothing from it is
kept or merged.

For unattended installs:

```bash
./install.sh --non-interactive --prompt-mode default
./install.sh --non-interactive --prompt-file ./my-prompt.txt
```

Without any prompt flag, a non-interactive first install uses the default prompt, and a non-interactive
re-run keeps the existing role.

## Usage

Single prompt:

```bash
viking "How do I find which process is listening on port 8080?"
```

Pipe logs, code or command output in as context:

```bash
journalctl -u ssh -n 50 --no-pager | viking "Anything suspicious here?"
cat script.py | viking "Find the bug"
dmesg | tail -n 40 | viking "Explain these errors"
```

Interactive chat (when your ShellGPT version supports it; the installer checks):

```bash
viking --interactive      # or just: viking
```

Press Ctrl+C to leave. The session is not stored on disk.

Other commands:

```bash
viking --viking-info      # model, prompt option, paths
viking --viking-help      # short help
viking --help             # full ShellGPT options
```

Anything else you pass is forwarded unchanged to ShellGPT, for example `viking --no-md "..."` for plain
text output.

Good to know:

- When `viking` receives piped input, it reads it as part of the prompt. In scripts without piped input,
  use `viking "..." </dev/null` so it does not wait on stdin.
- The launcher always selects the Viking role, so ShellGPT's own special modes (`-s` shell commands,
  `-c` code only) use the Viking prompt instead of their specialised ones. Use `sgpt` directly if you want
  those modes.
- Viking only produces text. It never runs commands for you.

## Customization

**Change the model later**

```bash
./install.sh --model llama3.2:3b --keep-prompt
```

Any tag from <https://ollama.com/library> works. Run `ollama list` to see what is installed. For a
single question with another installed model, without changing the default:

```bash
viking --model ollama_chat/llama3.2:3b "Your question"
```

**Update the Viking prompt later**

```bash
./install.sh            # choose option 1 or 2 again; Enter keeps your current model
```

or non-interactively:

```bash
./install.sh --non-interactive --prompt-file ./new-prompt.txt
```

**Upgrade ShellGPT and LiteLLM**

```bash
./install.sh --upgrade
```

**Give ShellGPT your full shell environment** (not recommended, see [Security notes](#security-notes)):

```bash
VIKING_INHERIT_ENV=1 viking "..."
```

## Verify the installation

```bash
command -v viking                 # should print ~/.local/bin/viking
viking --viking-info              # model, prompt option, paths
curl -s http://localhost:11434/api/version     # Ollama answers with a version
ollama list                       # your model is listed
viking "Reply with one short sentence." </dev/null     # end-to-end test
```

If `command -v viking` prints nothing, open a new terminal or run:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

## What gets installed where

| What | Where |
|---|---|
| Launcher | `~/.local/bin/viking` |
| Python environment (ShellGPT, LiteLLM) | `~/.local/share/vikingai/venv` |
| VikingAI settings (no secrets) | `~/.config/vikingai/config.env` (mode 600) |
| Viking role (your system prompt) | `viking.json` in ShellGPT's role folder, normally `~/.config/shell_gpt/roles/` (mode 600) |
| Conversation caches | `~/.cache/vikingai/` (mode 700) |
| Install log | `~/.local/state/vikingai/install.log` |
| Ollama and its models | System-wide. With the systemd service, models live under `/usr/share/ollama/.ollama/models` |

How ShellGPT is configured: ShellGPT's own config file (`~/.config/shell_gpt/.sgptrc`) is shared and cannot be
moved, and it may belong to an existing OpenAI setup. VikingAI therefore does not edit it. The launcher passes
the settings as environment variables, which ShellGPT gives priority over its config file:

| Variable | Value |
|---|---|
| `DEFAULT_MODEL` | `ollama_chat/<your model>` (uses Ollama's `/api/chat`, which supports a proper system message) |
| `API_BASE_URL` | `http://localhost:11434` |
| `USE_LITELLM` | `true` |
| `OPENAI_USE_FUNCTIONS` | `false` |
| `OPENAI_API_KEY` | a dummy value, no real key is ever needed or sent |

## Troubleshooting

**`viking: command not found`**
Open a new terminal, or run `export PATH="$HOME/.local/bin:$PATH"`. If you used `--no-modify-path`, add that
line to your shell's startup file yourself.

**`cannot reach Ollama at http://localhost:11434`**
```bash
sudo systemctl start ollama        # systemd systems
systemctl status ollama --no-pager
journalctl -u ollama -n 50 --no-pager
ollama serve                       # without systemd (containers, WSL); keep it running
```
Without systemd, Ollama does not start by itself after a reboot.

**`sudo` problems**
The installer needs sudo only for apt and for Ollama. If you have no sudo rights, ask an administrator to
install `curl ca-certificates python3 python3-venv zstd` and Ollama (<https://docs.ollama.com/linux>),
then run `./install.sh` again.

**Model download fails or the tag is "unknown"**
Check the spelling and tag at <https://ollama.com/library>. Use `name:tag` (for example `llama3.2:3b`), not
LiteLLM-style names like `ollama/llama3.2`. Make sure you have enough disk space, then run
`ollama pull <tag>` by hand to see the full error.

**pip or ShellGPT installation fails**
Read `~/.local/state/vikingai/install.log`. Typical causes: no network or proxy, Python older than 3.10, a full
disk. To start clean: `rm -rf ~/.local/share/vikingai/venv && ./install.sh`.

**The end-to-end test times out**
The installation is complete, but the model is slow on your hardware (common on a Raspberry Pi or a small VM,
and the first request loads the model into memory). Try `viking "hello"` again later or switch to a smaller
model. `--skip-test` skips this check.

**Answers are poor or very slow**
That is the model, not the installer. Small models are limited. Try a larger one if your RAM allows it.

**Corporate proxy**
Set `https_proxy` / `http_proxy` for the installer. The launcher automatically bypasses proxies for
`localhost`.

**Something is still wrong**
Run `./install.sh` again. It repairs missing pieces and tells you what failed. The log is
`~/.local/state/vikingai/install.log`.

## Uninstall

```bash
./install.sh --uninstall
```

This removes the VikingAI environment, the `viking` launcher, its config, cache and log files, the `viking`
role, and the PATH block the installer added to `~/.bashrc` / `~/.zshrc`. It does **not** remove Ollama,
downloaded models, or ShellGPT's own config.

Optional, to remove your model and Ollama as well:

```bash
ollama rm <model-tag>      # for example: ollama rm qwen2.5-coder:1.5b
```

For removing Ollama itself, follow the "Uninstall" section of <https://docs.ollama.com/linux>.

## Security notes

- Runs as a normal user. No `sudo pip`, no `chmod 777`, no `eval`. Your custom prompt is treated as plain data:
  it is never executed or passed through the shell.
- The only executable downloaded from outside your distribution is Ollama's official installer
  (`https://ollama.com/install.sh`). It is fetched to a temporary file, checked to be a shell script, and run only
  after you confirm. Ollama does not publish a checksum for that script, so you are trusting ollama.com.
- Python packages stay in a private virtual environment. **LiteLLM versions 1.82.7 and 1.82.8 were malicious
  releases on PyPI (credential stealers, March 2026).** The installer excludes them and aborts if either is
  present. Supply-chain risk remains for any unpinned dependency, so consider running VikingAI on a machine or
  account that does not hold sensitive credentials.
- By default the launcher runs ShellGPT/LiteLLM with a minimal environment, so variables such as `AWS_*`,
  `GITHUB_TOKEN` or `SSH_AUTH_SOCK` do not reach the Python process. `VIKING_INHERIT_ENV=1` turns this off.
- ShellGPT's default cache lives in shared `/tmp` folders. VikingAI uses `~/.cache/vikingai` (mode 700) instead.
- Ollama listens on `localhost` by default. Do not expose it to a network without authentication.
- Language models make mistakes. Read a command before you run it, especially anything destructive.

## Testing status

Tested: syntax (`bash -n`), ShellCheck (clean), and complete runs on Ubuntu 24.04 as a non-root user with real
ShellGPT 1.5.1 and LiteLLM 1.83.4, against a stand-in for the Ollama HTTP API. Checked: default and custom prompt
paths (the saved role equals the selected prompt byte for byte, including `$()`, backticks and `%s`), empty-prompt
handling, re-runs replacing the role, `--keep-prompt`, `--prompt-file`, idempotence, uninstall, input validation,
failure handling for bad model tags, 32-bit CPUs, root and missing sudo, and the request that reaches the model
(system prompt, dummy key, piped input).

**Not tested:** Ollama's real installer and service start, the apt installation path, real model inference, and
Kali, Debian and Raspberry Pi OS hardware. Please report problems you hit there.

## Manual install (other distributions)

The installer targets Debian-based systems, and this path is not covered by its tests. Roughly:

1. Install Ollama: <https://docs.ollama.com/linux>, then `ollama pull qwen2.5-coder:1.5b`.
2. Create a venv and install ShellGPT with LiteLLM, excluding the malicious releases:
   ```bash
   python3 -m venv ~/.local/share/vikingai/venv
   ~/.local/share/vikingai/venv/bin/pip install "shell-gpt[litellm]" "litellm!=1.82.7,!=1.82.8"
   ```
3. Create a role named `viking` with `sgpt --create-role viking`, then run ShellGPT with these environment
   variables set:
   ```bash
   DEFAULT_MODEL=ollama_chat/qwen2.5-coder:1.5b API_BASE_URL=http://localhost:11434 \
   USE_LITELLM=true OPENAI_USE_FUNCTIONS=false OPENAI_API_KEY=ollama-local \
   ~/.local/share/vikingai/venv/bin/sgpt --role viking "your question"
   ```
   Note that `--create-role` adds a "You are viking" line in front of your text. The installer avoids this by
   writing the role file itself.

## Acknowledgements

[Ollama](https://ollama.com), [ShellGPT](https://github.com/TheR1D/shell_gpt) and
[LiteLLM](https://github.com/BerriAI/litellm) do the heavy lifting.
