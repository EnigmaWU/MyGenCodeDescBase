# Fork & Safe Agent Workflow

This repository is a **base template** for all genCodeDesc forks. To ensure safe, reproducible, and isolated development with code agents (Copilot, LLMs, etc.), follow this workflow:

## 1. Fork This Repo

- Create your own fork (e.g., `MyGenCodeDesc_Copilot_GPT-5.4-Xhigh_Python`).

## 2. Open in VS Code

- Clone your fork locally.
- Open the folder in VS Code.

## 3. Reopen in Dev Container

- If prompted, click "Reopen in Container" (requires Docker Desktop running).
- Or: Open the Command Palette (⇧⌘P) → "Dev Containers: Reopen in Container".

## 4. Develop Safely

- All code agent work, builds, and tests run **inside the container**.
- The container uses a non-root user and does not mount host secrets or Docker socket by default.
- You can install Python, C++, Rust, and other tools as needed inside the container.

## 5. Commit & Push

- All changes are committed from inside the container.
- Push to your fork and open PRs as usual.

## Requirements

- **Docker Desktop** must be installed and running on your host (macOS, Windows, Linux).
- **VS Code** with the "Dev Containers" extension installed.

## Why?

- Keeps your host system clean and safe.
- Ensures all forks have a reproducible, isolated environment.
- Makes it easy for code agents to experiment without risk.

---

For more, see [README.md](README.md).
