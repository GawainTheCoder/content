---
title: 'Run Omni and Claude Engineer in Daytona'
description: 'Build repeatable Daytona sandboxes for Omni Engineer and Claude Engineer with Dev Containers, API keys, and a small coding task.'
date: 2026-05-11
author: 'Ashter Haider'
tags: ['Daytona', 'Dev Containers', 'AI Engineer']
---

# Run Omni and Claude Engineer in Daytona

AI coding assistants are easiest to evaluate when the runtime is disposable, repeatable, and separated from your laptop. That matters even more for tools that can read files, create files, install packages, search the web, or run shell commands.

In this guide, you will run [Omni Engineer](https://github.com/Doriandarko/omni-engineer) and [Claude Engineer](https://github.com/Doriandarko/claude-engineer) inside Daytona sandboxes.
You will also add a `.devcontainer/devcontainer.json` file to each project so the setup can be reused by other contributors instead of being rebuilt from memory.

The example task is intentionally small: ask an [AI engineer](/definitions/20260511_definition_ai_engineer.md) to create a Python command-line utility and then inspect the files it changes.
The goal is not to benchmark model quality. The goal is to show a clean workflow for using agentic developer consoles in a remote, recoverable environment.

![Architecture diagram showing a developer using Daytona to run Omni Engineer and Claude Engineer sandboxes](/assets/20260511_running_omni_and_claude_engineer_inside_daytona_img1.png)

## TL;DR

- Add a `.devcontainer/devcontainer.json` file to each AI engineer repository.
- Use Daytona to create an isolated sandbox, clone the repository, and run the setup commands.
- Keep provider API keys in environment variables or `.env`, not in committed files.
- Run Omni Engineer from the terminal and Claude Engineer from either its CLI or web UI.
- Use a small coding task first so you can verify file writes, diffs, and recovery before using the tools on larger projects.

## What You Will Build

You will create two repeatable environments:

| Project | Model provider | Primary run command | Useful Daytona workflow |
| --- | --- | --- | --- |
| Omni Engineer | OpenRouter | `python main.py` | Terminal-based file editing and multi-model experiments |
| Claude Engineer | Anthropic | `python ce3.py` or `python app.py` | CLI work or a browser-based UI on port `5000` |

Both projects are Python applications. A shared Dev Container pattern works well:

- Use the official Python 3.11 Dev Container image.
- Install Git in the container.
- Install Python dependencies from `requirements.txt`.
- Copy `.env.example` to `.env` if the developer has not created one.
- Pass API keys through environment variables when they are already present on the host.
- Configure VS Code Python defaults for people who open the sandbox in an editor.

This keeps the environment definition close to the code. It also makes the setup reviewable in a pull request, which is the right place to discuss project-specific defaults.
The companion PRs for this article are [Doriandarko/omni-engineer#26](https://github.com/Doriandarko/omni-engineer/pull/26) and [Doriandarko/claude-engineer#250](https://github.com/Doriandarko/claude-engineer/pull/250).

## Prerequisites

You need four things before starting:

- A Daytona account and API key from the [Daytona Dashboard](https://app.daytona.io).
- The Daytona CLI. The official CLI docs show `brew install daytonaio/cli/daytona` for macOS and Linux, and a PowerShell installer for Windows.
- A GitHub account so you can fork the two AI engineer repositories and open pull requests.
- API keys for the AI tools you want to run.

Omni Engineer uses OpenRouter. Add this value to its `.env` file:

```bash
OPENROUTER_API_KEY="your-openrouter-key"
```

Claude Engineer uses Anthropic, and optionally E2B for code execution:

```bash
ANTHROPIC_API_KEY="your-anthropic-key"
E2B_API_KEY="your-e2b-key"
```

Keep those values out of Git. The `.env.example` files are committed as templates; your real `.env` files stay local to the sandbox.

## Add a Dev Container to Omni Engineer

Fork `Doriandarko/omni-engineer`, clone your fork, and create a branch:

```bash
git clone https://github.com/<your-user>/omni-engineer.git
cd omni-engineer
git checkout -b add-daytona-devcontainer
mkdir -p .devcontainer
```

Create `.devcontainer/devcontainer.json`:

```json
{
  "name": "Omni Engineer",
  "image": "mcr.microsoft.com/devcontainers/python:1-3.11-bookworm",
  "features": {
    "ghcr.io/devcontainers/features/git:1": {}
  },
  "postCreateCommand": "python -m pip install --upgrade pip && pip install -r requirements.txt && if [ ! -f .env ]; then cp .env.example .env; fi",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-python.vscode-pylance"
      ],
      "settings": {
        "python.defaultInterpreterPath": "/usr/local/bin/python"
      }
    }
  },
  "remoteEnv": {
    "OPENROUTER_API_KEY": "${localEnv:OPENROUTER_API_KEY}",
    "PYTHONUNBUFFERED": "1"
  }
}
```

This is a small file, but it removes several setup decisions. Everyone gets Python 3.11, the same dependency install command, and a generated `.env` file they can edit without touching the repository's committed template.
The `remoteEnv` line also supports the common case where the developer exports `OPENROUTER_API_KEY` before creating the sandbox.

Commit it:

```bash
git add .devcontainer/devcontainer.json
git commit -m "Add Daytona devcontainer"
```

## Add a Dev Container to Claude Engineer

Repeat the same pattern for `Doriandarko/claude-engineer`:

```bash
git clone https://github.com/<your-user>/claude-engineer.git
cd claude-engineer
git checkout -b add-daytona-devcontainer
mkdir -p .devcontainer
```

Create `.devcontainer/devcontainer.json`:

```json
{
  "name": "Claude Engineer",
  "image": "mcr.microsoft.com/devcontainers/python:1-3.11-bookworm",
  "features": {
    "ghcr.io/devcontainers/features/git:1": {}
  },
  "postCreateCommand": "python -m pip install --upgrade pip && pip install -r requirements.txt && if [ ! -f .env ]; then cp .env.example .env; fi",
  "forwardPorts": [
    5000
  ],
  "portsAttributes": {
    "5000": {
      "label": "Claude Engineer web UI",
      "onAutoForward": "notify"
    }
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-python.vscode-pylance"
      ],
      "settings": {
        "python.defaultInterpreterPath": "/usr/local/bin/python"
      }
    }
  },
  "remoteEnv": {
    "ANTHROPIC_API_KEY": "${localEnv:ANTHROPIC_API_KEY}",
    "E2B_API_KEY": "${localEnv:E2B_API_KEY}",
    "PYTHONUNBUFFERED": "1"
  }
}
```

The main difference is `forwardPorts`. Claude Engineer includes a Flask web UI on port `5000`, so the Dev Container metadata tells compatible tools that this port is expected.
The Anthropic and E2B keys are passed through when they are already present on the host, while the copied `.env.example` still gives readers a concrete file to edit inside the sandbox.

Commit the change:

```bash
git add .devcontainer/devcontainer.json
git commit -m "Add Daytona devcontainer"
```

Open pull requests against both upstream repositories. In the Daytona content pull request, link to those two PRs so reviewers can verify that the article is backed by real environment contributions.

## Create a Daytona Sandbox

The Daytona CLI can create a sandbox directly:

```bash
daytona create --name omni-engineer
```

Clone your fork into the sandbox and check the repository status:

```bash
daytona exec omni-engineer -- git clone https://github.com/<your-user>/omni-engineer.git /home/daytona/omni-engineer
daytona exec omni-engineer --cwd /home/daytona/omni-engineer -- git status --short
```

If your Daytona workflow automatically consumes Dev Container metadata, the `postCreateCommand` handles dependency installation. If you are using a plain sandbox, run the same setup command explicitly:

```bash
daytona exec omni-engineer --cwd /home/daytona/omni-engineer -- python -m pip install --upgrade pip
daytona exec omni-engineer --cwd /home/daytona/omni-engineer -- python -m pip install -r requirements.txt
daytona exec omni-engineer --cwd /home/daytona/omni-engineer -- cp .env.example .env
```

Open an SSH session when you want an interactive terminal:

```bash
daytona ssh omni-engineer
cd /home/daytona/omni-engineer
```

Add your OpenRouter key to `.env`:

```bash
python - <<'PY'
from pathlib import Path

env = Path(".env")
current = env.read_text() if env.exists() else ""
lines = [line for line in current.splitlines() if not line.startswith("OPENROUTER_API_KEY=")]
lines.append('OPENROUTER_API_KEY="your-openrouter-key"')
env.write_text("\n".join(lines) + "\n")
PY
```

Run Omni Engineer:

```bash
python main.py
```

The current repository contains `main.py` as the runnable entrypoint. If a README or older branch mentions `omni-eng.py`, prefer the file that exists in the checked-out branch.

Now try a small prompt:

```text
/new tools/slugify.py
Create a Python function called slugify that lowercases a title, replaces whitespace with hyphens, removes punctuation, and includes a small __main__ demo.
```

Then inspect the result from another terminal:

```bash
daytona exec omni-engineer --cwd /home/daytona/omni-engineer -- git diff -- tools/slugify.py
```

That final command is the important habit. Treat agent output like any other code contribution: review the diff, run the file, and keep the sandbox disposable.

## Run Claude Engineer in Daytona

Create a second sandbox:

```bash
daytona create --name claude-engineer
daytona exec claude-engineer -- git clone https://github.com/<your-user>/claude-engineer.git /home/daytona/claude-engineer
daytona exec claude-engineer --cwd /home/daytona/claude-engineer -- python -m pip install --upgrade pip
daytona exec claude-engineer --cwd /home/daytona/claude-engineer -- python -m pip install -r requirements.txt
daytona exec claude-engineer --cwd /home/daytona/claude-engineer -- cp .env.example .env
```

SSH into it:

```bash
daytona ssh claude-engineer
cd /home/daytona/claude-engineer
```

Edit `.env`:

```bash
python - <<'PY'
from pathlib import Path

env = Path(".env")
current = env.read_text() if env.exists() else ""
lines = [line for line in current.splitlines() if not line.startswith(("ANTHROPIC_API_KEY=", "E2B_API_KEY="))]
lines.append("ANTHROPIC_API_KEY=your-anthropic-key")
lines.append("E2B_API_KEY=your-e2b-key")
env.write_text("\n".join(lines) + "\n")
PY
```

Run the CLI:

```bash
python ce3.py
```

Use a constrained first task:

```text
Create a file named examples/readme_summary.py that prints the first Markdown heading from readme.md. Keep it dependency-free.
```

You can also run the web UI:

```bash
python app.py
```

In a separate shell, request a Daytona preview URL for port `5000`:

```bash
daytona preview-url claude-engineer --port 5000
```

Open the returned URL, send the same small task, and compare the resulting Git diff:

```bash
daytona exec claude-engineer --cwd /home/daytona/claude-engineer -- git diff -- examples/readme_summary.py
```

## What to Check Before Trusting the Agent

Use this checklist before giving either tool a larger job:

| Check | Why it matters | Command |
| --- | --- | --- |
| Dependencies installed | Confirms the sandbox matches the repository | `python -m pip check` |
| API key loaded | Avoids confusing provider errors | `python -c "import os; print(bool(os.getenv('ANTHROPIC_API_KEY') or os.getenv('OPENROUTER_API_KEY')))"` |
| Git diff is small | Makes review possible | `git diff --stat` |
| Generated code runs | Catches syntax and dependency mistakes | `python path/to/generated_file.py` |
| Sandbox can be stopped | Keeps spend and state under control | `daytona stop <sandbox-name>` |

For real project work, keep tasks narrow. Instead of "refactor this repo," ask for one file, one behavior, and one test. Daytona gives you the safety net, but you still need a review loop.

You can also verify the repository assumptions before starting either assistant:

```bash
daytona exec omni-engineer --cwd /home/daytona/omni-engineer -- test -f main.py
daytona exec omni-engineer --cwd /home/daytona/omni-engineer -- test -f .env.example
daytona exec claude-engineer --cwd /home/daytona/claude-engineer -- test -f ce3.py
daytona exec claude-engineer --cwd /home/daytona/claude-engineer -- test -f app.py
daytona exec claude-engineer --cwd /home/daytona/claude-engineer -- test -f requirements.txt
```

## Troubleshooting

**The assistant starts but API calls fail.** Check `.env`, then confirm the process sees the value:

```bash
python -c "import os; print(os.getenv('OPENROUTER_API_KEY') is not None)"
python -c "import os; print(os.getenv('ANTHROPIC_API_KEY') is not None)"
```

**Claude Engineer's web UI is not reachable.** Confirm Flask is running on port `5000`, then request a fresh preview URL:

```bash
daytona ssh claude-engineer
python - <<'PY'
import socket

s = socket.socket()
print(s.connect_ex(('127.0.0.1', 5000)) == 0)
PY
exit
daytona preview-url claude-engineer --port 5000
```

**A generated file is wrong.** Do not edit blindly inside the conversation. Check the diff first:

```bash
git diff
```

If the result is not useful, reset only the generated file:

```bash
git checkout -- path/to/generated_file.py
```

That is another reason to run these tools in a sandbox: you can stop, archive, or delete the environment when an experiment gets messy.

## Conclusion

Omni Engineer and Claude Engineer both become easier to evaluate when their setup is encoded as Dev Container metadata and run inside Daytona. The repository keeps its own environment instructions, Daytona provides an isolated sandbox, and Git gives you a reviewable record of what the agent changed.

Start with one small file-generation task, inspect the diff, and only then move to larger changes. That workflow gives you the practical upside of AI developer agents without turning your local machine into the experiment.

## References

- [Daytona Getting Started](https://www.daytona.io/docs/getting-started)
- [Daytona CLI reference](https://www.daytona.io/docs/en/tools/cli/)
- [Dev Container documentation](https://code.visualstudio.com/docs/devcontainers/create-dev-container)
- [Omni Engineer repository](https://github.com/Doriandarko/omni-engineer)
- [Claude Engineer repository](https://github.com/Doriandarko/claude-engineer)
- [Omni Engineer Daytona Dev Container PR](https://github.com/Doriandarko/omni-engineer/pull/26)
- [Claude Engineer Daytona Dev Container PR](https://github.com/Doriandarko/claude-engineer/pull/250)
