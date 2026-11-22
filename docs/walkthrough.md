# Walkthrough

What each role does, task by task, and why. The roles run in the order of `site.yml`; each has a tag to run it alone.

Before the roles, two checks stop the run early:

- **Zorin OS 18 / Ubuntu 24.04**: package names and versions are those of Ubuntu noble, which Zorin OS 18 is built on. The check reads `/etc/os-release` (`ID`/`ID_LIKE` must include `ubuntu`, `UBUNTU_CODENAME` must be `noble`) instead of Ansible's distribution facts: Ansible does not list Zorin OS among the Debian derivatives, so on Zorin its `os_family` reads `Zorin OS`.
- **Not root**: the playbook installs into the home directory of the user who runs it. Run with `sudo`, it would install under `/root` and the service would belong to the wrong user. Root is obtained per task with `become: true`, which is why the playbook is run with `-K`.

## gpu_vulkan (tag `gpu`)

The RX 5700 XT is RDNA 1 (Navi 10), not supported by ROCm. llama.cpp talks to it through Vulkan, using Mesa's open-source RADV driver.

| Task | What and why |
| --- | --- |
| Install the Vulkan driver and tools | `mesa-vulkan-drivers` (RADV), `libvulkan1` (loader), `vulkan-tools` (`vulkaninfo`), `radeontop` (live GPU/VRAM usage, useful during the benchmarks). On a Zorin desktop most of these are already there. |
| Add the user to the GPU groups | `render` and `video` own `/dev/dri/*`. A desktop session already has access through logind, but the groups also cover the systemd user service started outside the session and SSH logins. A new group counts from the next login. |
| List the Vulkan devices / check RADV | `vulkaninfo --summary` must show an AMD device on RADV (`AMD Radeon RX 5700 XT (RADV NAVI10)`). `llvmpipe`, the CPU fallback, is always listed and does not count. The check fails the run, so nothing gets built against a missing GPU. |

## llama_cpp (tag `llama-cpp`)

llama.cpp is built from source at a pinned release (`llama_cpp_version`, a `b<number>` tag): Ubuntu has no Vulkan build, and a pinned tag makes every run produce the same binaries.

| Task | What and why |
| --- | --- |
| Install the build dependencies | compiler, CMake, git, and for Vulkan `libvulkan-dev`, `glslc` (compiles the shaders) and `spirv-headers` (required by the Vulkan backend, not pulled in by `libvulkan-dev`). `libssl-dev` gives llama.cpp HTTPS. |
| Download the sources | shallow clone of the tag into `~/.local/src/llama.cpp-<tag>`, one directory per version. |
| Configure the build | `-DGGML_VULKAN=ON`, Release, and `-DBUILD_SHARED_LIBS=OFF`: static binaries are self-contained, so they can be copied out of the build tree and an older version keeps working next to a newer one. Tests and examples are not built. |
| Build | only `llama-server`, `llama-bench`, `llama-cli`. Compiling the Vulkan shaders makes the first build take 10–20 minutes; `creates:` skips it on later runs. |
| Install the binaries / point "current" | copied to `~/.local/opt/llama.cpp/<tag>/bin`, then `~/.local/opt/llama.cpp/current` is moved to this version. Rolling back means moving the link back. A running llama-swap is restarted so new models start with the new binaries. |
| Link the programs | `llama-server`, `llama-bench`, `llama-cli` in `~/.local/bin`, for use from the terminal. |
| Check that it runs | `llama-server --version`, printed at the end of the role. |

## models (tag `models`)

The models are GGUF files from Hugging Face, listed in `ai_models` (`inventory/group_vars/all/main.yml`).

| Task | What and why |
| --- | --- |
| List the files of each repository | Hugging Face API (`/api/models/<repo>/tree/main`): file names, sizes and the SHA-256 of each file. |
| Find the candidate files / check exactly one | a model names either its exact `file` or a `quant` such as `Q4_K_M`; the file whose name ends with `-Q4_K_M.gguf` is picked. Unsloth's `UD-` variants share the suffix and are excluded unless asked for (`quant: UD-Q4_K_M`); `mmproj` files (vision add-ons) are never picked. No match or more than one stops the run with the list of candidates, rather than downloading the wrong 20 GB. |
| Download (`download.yml`, one model at a time) | with `curl` into `<name>.part` next to its final place. `--continue-at -` resumes from what is already there, so an interrupted download (dropped connection, playbook stopped, computer turned off) continues on the next run instead of starting over; `--retry` resumes dropped connections within the same run. The growing `.part` file shows the progress. |
| Check the size, then the SHA-256 | a short file is an interrupted download: the run stops and the next one resumes. A complete file whose SHA-256 differs from the one Hugging Face publishes is corrupt: it is deleted and downloaded again on the next run. Only a verified file is moved to its final name. |
| Record the verified checksum | `<name>.sha256` next to the model: later runs compare it and the size instead of hashing tens of GB again, so they skip the model in a second. |
| Write the list of downloaded models | `~/.config/ai/models.json`: id, name, local path, context size, extra arguments. llama_swap builds its configuration from it, so the two roles can run separately. |

Models with `download: false` are skipped and do not appear in llama-swap. Files of models removed from the list stay on disk (see maintenance).

## llama_swap (tag `llama-swap`)

| Task | What and why |
| --- | --- |
| Download and unpack the release | the `linux_amd64` archive of the pinned version, checked against the SHA-256 published with the release, unpacked into `~/.local/opt/llama-swap/<version>`; `current` points to it, `~/.local/bin/llama-swap` to `current`. |
| Write the configuration | `~/.config/ai/llama-swap.yaml` from the template and `models.json`, checked with `llama-swap -validate` before it replaces the old one. One entry per model: the `llama-server` command line with the model file, its context size and the common arguments (`llama_swap_server_args`): chat template from the GGUF (`--jinja`), flash attention, KV cache at 8 bits (half the memory of f16, so longer contexts fit), one conversation at a time. `globalTTL` unloads a model after `ai_model_ttl` seconds without requests. |
| Install the systemd user service | `~/.config/systemd/user/llama-swap.service`: `llama-swap -listen 127.0.0.1:8080 -watch-config`. Loopback only. `-watch-config` reloads the configuration when the file changes, so adding a model needs no restart. |
| Start at login, or not | disabled by default (`llama_swap_autostart: false`): it starts with `ai on`, `ai chat` or the menu entry. |

Handlers: `daemon-reload` after the unit changes; `try-restart` when the binaries or the unit change, which restarts the service only if it is running.

### Memory while idle

llama-swap itself uses a few MB and no VRAM. A model is loaded on the first request for it and unloaded after `ai_model_ttl` seconds (600) without requests, or right away with `ai unload` / `ai off`. So even a forgotten `ai on` gives the memory back by itself.

## launcher (tag `launcher`)

| Task | What and why |
| --- | --- |
| Install the ai command | `~/.local/bin/ai`: `on`, `off`, `status`, `chat [model]`, `ui`, `models`, `unload`, `logs`. `chat` starts the service if needed and opens `http://127.0.0.1:8080/upstream/<model>/`, the chat page of that model's `llama-server`; the first request loads the model. |
| Install the desktop launchers | **AI locale** (`ai chat`) and **AI locale: spegni** (`ai off`) in the applications menu. |
