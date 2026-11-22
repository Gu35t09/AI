# Maintenance

Everything is changed through the repository and the playbook: edit, run, commit. Files written by the playbook (`~/.config/ai/*`, the systemd unit, the `ai` command) are overwritten on the next run.

## Calendar

| When | What |
| --- | --- |
| Monthly, or when a model needs it | update llama.cpp (new model architectures and speed-ups land there first) |
| Every few months | update llama-swap |
| When a better model comes out | add it next to the current one, compare, then remove the old one |
| Zorin updates | nothing to do: Mesa updates come with the system and llama.cpp does not depend on its version |

## Update llama.cpp

1. Pick a release on https://github.com/ggml-org/llama.cpp/releases (tags `b<number>`).
2. Change `llama_cpp_version` in `roles/llama_cpp/defaults/main.yml`.
3. Build and switch:

   ```bash
   ansible-playbook site.yml -K --tags llama-cpp
   ```

   The new version is built next to the old one; `current` moves only after the build succeeds, and a running service is restarted.
4. Check (`verify.md`, llama.cpp section), commit.

**Roll back**: set the previous tag and run the same command (nothing is rebuilt, the link moves back), or by hand:

```bash
ln -sfn ~/.local/opt/llama.cpp/<previous tag> ~/.local/opt/llama.cpp/current
systemctl --user try-restart llama-swap
```

**Free disk space**: each version takes about 1 GB of sources and build plus a few hundred MB installed. Once the new one has proven itself:

```bash
rm -rf ~/.local/src/llama.cpp-<old tag> ~/.local/opt/llama.cpp/<old tag>
```

## Update llama-swap

1. Pick a release on https://github.com/mostlygeek/llama-swap/releases.
2. In `roles/llama_swap/defaults/main.yml` change `llama_swap_version` (without the `v`) and `llama_swap_sha256`, the `linux_amd64` line of `llama-swap_<version>_checksums.txt` in the same release.
3. `ansible-playbook site.yml --tags llama-swap`, check, commit. Old versions stay in `~/.local/opt/llama-swap/` and can be deleted.

## Models

### Add a model

1. Find a GGUF repository on Hugging Face (e.g. by unsloth, bartowski, or the model's authors). Check the license on the model page.
2. Add an entry to `ai_models` in `inventory/group_vars/all/main.yml`:

   ```yaml
   - id: "short-name"            # used in "ai chat short-name" and in the API
     name: "Display name"
     hf_repo: "owner/Repository-GGUF"
     quant: "Q4_K_M"             # or file: "exact/path.gguf"
     ctx_size: 32768
   ```

3. `ansible-playbook site.yml --tags models,llama-swap` (no sudo needed). The running service picks up the new configuration by itself.

How big a model fits: as a rule of thumb, at Q4_K_M a model takes about 0.6 GB per billion parameters, plus the KV cache. Up to ~8–9B dense fits in the 8 GB of VRAM; mixture-of-experts models up to ~35B total fit in VRAM + RAM; keep at least 6–8 GB of RAM free for the desktop.

### Remove a model

Set `download: false` (or delete the entry), run `--tags models,llama-swap`, then delete the file from `~/.local/share/ai/models/`: the playbook never deletes models.

### Change context size or arguments

Edit `ctx_size` or `args` of the model, then `--tags models,llama-swap`. A longer context uses more memory: if the model no longer loads, `ai logs` shows the allocation error.

## Change port or idle time

`ai_listen_port` and `ai_model_ttl` in `inventory/group_vars/all/main.yml`, then:

```bash
ansible-playbook site.yml --tags llama-swap,launcher
```

## Troubleshooting

| Symptom | Where to look |
| --- | --- |
| `ai on` says llama-swap does not answer | `ai logs`; `systemctl --user status llama-swap` |
| The chat stays on "loading" | the model is loading: large models take tens of seconds from disk the first time. `ai logs` shows the progress and any error. |
| A model fails to load (out of memory) | close games or other GPU programs; lower `ctx_size`; check free memory with `free -h` and `radeontop` |
| Very slow generation | check that the GPU is used: the model log (`ai logs`) must show the Vulkan device `RADV NAVI10` and layers offloaded to it; a dense model larger than ~9B is slow by nature |
| `vulkaninfo` shows only `llvmpipe` | Mesa driver missing or broken: `sudo apt install --reinstall mesa-vulkan-drivers`, reboot |
| Permission denied on `/dev/dri/renderD128` | log out and back in after the playbook added you to `render`/`video`; check with `groups` |
| The playbook stops at "exactly one file" | the repository has none or several files for that quantization: copy the exact path from the message into `file:` |
| systemd errors in the playbook ("Failed to connect to bus") | run the playbook from the desktop session (or a login of the same user), not with `sudo` and not from a `su` shell |

## Remove everything

```bash
ai off
systemctl --user disable llama-swap
rm -f ~/.config/systemd/user/llama-swap.service ~/.local/bin/{ai,llama-swap,llama-server,llama-bench,llama-cli} \
      ~/.local/share/applications/ai-{chat,stop}.desktop
rm -rf ~/.config/ai ~/.local/opt/llama.cpp ~/.local/opt/llama-swap ~/.local/src/llama.cpp-*
rm -rf ~/.local/share/ai          # the models: tens of GB
systemctl --user daemon-reload
```

The system packages (Vulkan driver and tools, build dependencies) stay; they are ordinary Ubuntu packages.
