# Notes for Claude

## Contribution workflow

Claude contributes to this repository as an external contributor, through the GitHub account `v-claude-gh` and its fork `v-claude-gh/AI`. The account has no write access to `Gu35t09/AI`, on purpose.

1. Work in the fork. If the session only has `Gu35t09/AI`, attach the fork with `add_repo` (owner `v-claude-gh`, access `push`).
2. Before starting, bring the fork's `main` up to date with upstream: add `https://github.com/Gu35t09/AI.git` as the `upstream` remote, fetch `main` with a depth (e.g. `--depth=50`), fast-forward the fork's `main` to `upstream/main` and push it to `origin`.
3. Make the change on a new branch `claude/<short-description>` created from that `main`, commit with a message that can serve as the pull request's title and description, and push the branch to the fork.
4. Do not try to open the pull request: this session's proxy allows the GitHub API only on repositories attached with push access, so creating a PR on `Gu35t09/AI` is refused. Give the owner the compare link instead, and they open the PR:
   `https://github.com/Gu35t09/AI/compare/main...v-claude-gh:AI:<branch>`
5. To check whether a PR was merged, fetch `upstream/main` and look for the branch's commit (`git merge-base --is-ancestor <commit> upstream/main`). Then fast-forward the fork's `main` again.

Old branches on the fork can stay; there is no need to delete them.

## Repository rules

- Everything must be open source under an OSI-approved license; the repository is GPL-3.0. Do not add dependencies under GPL-incompatible or non-OSI licenses (e.g. LM Studio, Open WebUI's license); AGPL libraries such as PyMuPDF are avoided.
- The target is a desktop computer the owner uses every day (Zorin OS 18, Ubuntu 24.04 base; Radeon RX 5700 XT, 8 GB VRAM, Vulkan only, no ROCm). The playbook runs as the user, with `become: true` only on tasks that need root; nothing listens outside loopback.
- Pin versions with checksums (llama.cpp tag, llama-swap version and SHA-256, model SHA-256 from Hugging Face).
- Keep `README.md`, `docs/walkthrough.md`, `docs/maintenance.md` and `verify.md` in step with the roles. Repository files are in English; user-facing messages of the `ai` command and the menu entries are in Italian.
- The owner's computer is not reachable from the cloud session: test what can be tested in the container (Ubuntu 24.04), and give the owner the commands for the rest.
