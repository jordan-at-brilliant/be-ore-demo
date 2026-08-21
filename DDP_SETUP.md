# Distributed Training Setup — RTX 5080 + RTX 4060 Ti

Two Windows machines training the same model simultaneously using PyTorch
Distributed Data Parallel (DDP). Expected speedup: ~1.7–1.9× vs the 5080 alone.

Both GPUs have 16GB VRAM so the setup is identical on both.

---

## How it works

Each machine loads the full model and a different shard of the training data.
After every step, they average their gradients over the network. From the
model's perspective it's like training with double the batch size on one machine.

---

## One-time setup — 4060 Ti machine (do this once)

### 1. Install Python 3.11

Download from **https://python.org/downloads** — pick Python 3.11.x (not 3.12+).

During install: **check "Add Python to PATH"**

Verify in cmd:
```
python --version
```

### 2. Install Git

Download from **https://git-scm.com** — use all defaults.

Verify:
```
git --version
```

### 3. Authenticate GitHub

The repo is private. You need to auth with GitHub CLI or a personal access token.

**Option A — GitHub CLI (easiest):**
```
winget install GitHub.cli
gh auth login
```
Choose GitHub.com → HTTPS → Login with a web browser → follow the prompts.

**Option B — Personal access token:**
- Go to github.com → Settings → Developer settings → Personal access tokens
- Create a token with `repo` scope
- When git asks for a password, paste the token

### 4. Clone the repo

```
cd C:\Users\YourName\Documents
git clone https://github.com/BrilliantEarth/be-sage.git
cd be-sage
```

### 5. Run the setup script

From the repo root, right-click `scripts\setup_ddp_node1.bat` → **Run as Administrator**.

This installs:
- `transformers`, `peft`, `trl`, `datasets`, `accelerate`
- `bitsandbytes` (8-bit Adam optimizer)
- PyTorch with CUDA (nightly cu128 build — required for the 4060 Ti's CUDA version)

It also opens port 29500 in Windows Firewall for DDP communication.

---

## Network setup — both machines

Both machines need to be on the **same LAN** (home router or office switch).
A direct ethernet connection between them is ideal but WiFi works.

### Find the primary machine's LAN IP

On the **5080 machine**, open cmd:
```
ipconfig
```
Look for `IPv4 Address` under the active adapter (Ethernet or WiFi).
Example: `192.168.1.10`

Write this down — you'll put it in both batch files.

### Open port 29500 on the 5080 machine (one-time, run as Administrator)

```
netsh advfirewall firewall add rule name="BE Sage DDP" dir=in action=allow protocol=TCP localport=29500
```

The setup script already does this on the 4060 Ti.

---

## Configure the batch files

**On the 5080 machine** — edit `scripts\launch_ddp_node0.bat`:
```
set MASTER_ADDR=192.168.1.10     <- replace with the 5080's actual LAN IP
```

**On the 4060 Ti machine** — edit `scripts\launch_ddp_node1.bat`:
```
set MASTER_ADDR=192.168.1.10     <- same IP as above (the 5080 machine)
```

---

## Running distributed training

### Step 1 — Sync training data on both machines

On both machines:
```
git pull
```

Both machines need `output\v2\training\qa_sample.jsonl` and
`output\v2\training\data\train.jsonl` / `valid.jsonl`.
Node 0 generates the splits if they don't exist. Node 1 reads them from disk.

### Step 2 — Start node 0 (5080 machine)

```
scripts\launch_ddp_node0.bat
```

It prints `Waiting for node 1 to connect...` and blocks.

### Step 3 — Start node 1 (4060 Ti machine) within ~5 minutes

```
scripts\launch_ddp_node1.bat
```

Training begins on both machines simultaneously. You'll see progress on both screens.

### Step 4 — After training

Adapters are saved on node 0 (5080 machine) only at:
```
output\v2\training\adapters\best_adapters\
```

Commit and push from the 5080 machine:
```
git add output\v2\training\adapters\best_adapters\
git commit -m "chore: update trained adapters (DDP)"
git push
```

Pull on Mac and run the eval harness:
```
git pull
python core/ml/eval_adapter.py
```

---

## Passing arguments

Arguments after the batch file name are forwarded to the training script.

```
scripts\launch_ddp_node0.bat --epochs 2 --batch-size 8
scripts\launch_ddp_node1.bat --epochs 2 --batch-size 8
```

Both machines must use the same arguments.

```
scripts\launch_ddp_node0.bat --delta
scripts\launch_ddp_node1.bat --delta
```

`--delta` runs 1 epoch, attention-only LoRA, 512-token cap (feedback queue retrains).

---

## Memory profile — both GPUs

| Component | Memory |
|-----------|--------|
| Llama 3.2 3B (bf16) | ~6.5 GB |
| LoRA adapters (r=8, 4 modules) | ~0.1 GB |
| Activations (batch 4, len 512) | ~2–3 GB |
| AdamW 8-bit optimizer | ~1–2 GB |
| **Total** | **~10–12 GB** |

Both the 5080 and 4060 Ti have 16GB, leaving 4–6 GB headroom.
If you hit OOM, reduce `--batch-size` to 2 (effective global batch stays 4 with 2 nodes).

---

## Troubleshooting

**"Connection refused" / timeout on node 1:**
- Check that the 5080's LAN IP in `launch_ddp_node1.bat` is correct
- Verify port 29500 is open on the 5080: `netsh advfirewall firewall show rule name="BE Sage DDP"`
- Make sure node 0 is already running and waiting before starting node 1
- Both machines must be on the same network (not one on VPN)

**"CUDA not available":**
- Reinstall PyTorch nightly: `pip install --pre torch --index-url https://download.pytorch.org/whl/nightly/cu128`
- Verify GPU: `python -c "import torch; print(torch.cuda.get_device_name(0))"`

**"Out of memory":**
- Reduce batch size: add `--batch-size 2` to both batch files
- Or reduce sequence length: add `--max-length 512` (already default for `--delta`)

**Training is slower than expected:**
- Check network speed between machines: `ping 192.168.1.10 -n 20`
- Ethernet is much better than WiFi for gradient sync
- Gradient sync adds ~5–15% overhead vs single-machine — this is normal

**Node 1 finishes but node 0 is still going (or vice versa):**
- This means one machine is faster — DDP synchronizes every step so the slower
  machine sets the pace. This is expected behavior.
