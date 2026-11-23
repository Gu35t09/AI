# Verification

Commands to check each part after the playbook has run. Run them as your own user in a terminal of the desktop session.

## 1. Playbook

```bash
ansible-playbook site.yml -K
```

A second run right after the first must end with `changed=0`: every task is idempotent.

## 2. GPU and Vulkan

```bash
vulkaninfo --summary | grep -E "deviceName|driverName"
```

Expected: a device `AMD Radeon RX 5700 XT (RADV NAVI10)` with driver `radv` (next to `llvmpipe`, the CPU fallback, which is always listed).

```bash
groups
```

Must include `render` and `video` (after logging out and back in, if the playbook added them).

## 3. llama.cpp

```bash
llama-server --version
llama-bench --list-devices
```

The version line shows the commit of the tag in `roles/llama_cpp/defaults/main.yml` (the build number reads 1 because the sources are a shallow clone); the device list shows `Vulkan0: AMD Radeon RX 5700 XT (RADV NAVI10)` with about 8 GB.

## 4. Models

```bash
ls -lh ~/.local/share/ai/models/
cat ~/.config/ai/models.json
```

One `.gguf` file per model with `download: true`, each with a `.gguf.sha256` next to it (written only after the SHA-256 matched) and no `.part` file left over.

## 5. llama-swap

```bash
llama-swap -validate -config ~/.config/ai/llama-swap.yaml && echo valid
ai on
ai status
ai models
```

`ai status` reports llama-swap active and no model loaded; `ai models` lists the model ids.

Only loopback listens:

```bash
ss -ltnp | grep llama-swap
```

Expected: `127.0.0.1:8080`, nothing on `0.0.0.0` or `*`.

## 6. A request through the API

```bash
curl -s http://127.0.0.1:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3.5-9b","messages":[{"role":"user","content":"Rispondi solo: ok"}]}' \
  | python3 -c 'import json,sys; r=json.load(sys.stdin); print(r["choices"][0]["message"]["content"]); print(r.get("timings"))'
```

The first request loads the model (seconds to a minute). `timings` shows the prompt and generation speed in tokens per second.

While the model is loaded, in another terminal:

```bash
radeontop
```

VRAM usage goes up by several GB: the model is on the GPU. The model log confirms it:

```bash
ai logs
```

Look for the Vulkan device and the number of layers offloaded to it.

## 7. Unloading

```bash
ai unload
ai status
```

No model loaded; `radeontop` shows the VRAM back to the desktop's own usage. Without `ai unload`, the same happens by itself after `ai_model_ttl` seconds (600) without requests.

```bash
ai off
ai status
```

`llama-swap spento`.

## 8. Chat and menu

- `ai chat` opens the browser on the chat of the default model; the first message loads the model.
- `ai ui` opens the llama-swap page (models, logs, playground).
- The applications menu has **AI locale** and **AI locale: spegni**.
