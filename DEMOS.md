# Local demo notes

This independent copy originates from [Divyansh Agrawal's Qwenputer](https://github.com/divagr18/qwenputer).
The original MIT license and citation are retained. Model weights remain separately distributed
under Apache-2.0; they are not included in this repository.

## Environment

- RTX 3090, 24 GB; NVIDIA driver 535.309.01.
- `experimental/riscv` is a self-contained package; run its commands from that directory.
- Its uv configuration selects the PyTorch CUDA 12.6 index instead of the default CUDA 13 build.
- Upstream code: `bec1a0c9654ac8151b6f36ed0154a19c53141384`.
- Model revision: `2d9068508f2b680badb7abcded96fdd107e9ba5d`.

```bash
cd experimental/riscv
uv sync --extra hf
uv tool run --from huggingface_hub hf download divagr1925/Qwen3.5-4B-Qwenputer \
  --revision 2d9068508f2b680badb7abcded96fdd107e9ba5d --local-dir qwenputer-rv
```

On the development server `qwenputer-rv` is a symlink into `/data/lyk/qwenputer/models/`.
The virtual environment and uv cache also live under `/data/lyk/qwenputer/` to avoid NFS I/O.
For subsequent installs on this server use `uv sync --extra hf --cache-dir /data/lyk/qwenputer/uv-cache`.
Keep checkpoints, caches and local results outside version control.

## Reference firmware demos

These run the project's reference RISC-V machine on the host. They validate the firmware,
but do not demonstrate inference through the Qwen model's weights.

```bash
uv run python -m chipformer.rv.calc --expr '123456789 * 987654321'
uv run python -m chipformer.rv.calc --expr 'gcd(1234567890,9876543210)'
uv run python -m chipformer.rv.calc --expr '(2**64 - 1) // 12345'
```

Observed on 2026-11-04:

| Demo | Output | Reference check |
| --- | --- | --- |
| Large multiplication | `121932631112635269` | True |
| GCD | `90` | True |
| Integer division | `1494268454735484` | True |
| C qsort, 1000 values | `err=0 min=4940 max=16772127 hash=72e4eb11` | Matches stored qemu output |
| C prime sieve | `primes below 10000: 1229` | Matches stored qemu output |
| MicroPython `mpy-s1` | `1267650600228229401496703205376` | Matches stored qemu output |

The C and MicroPython ELF files are already under `tests/data/rv/`.
The qemu comparison uses upstream's stored `qemu.json`; it does not invoke a newly installed qemu.

## Model inference

```bash
uv run python -m chipformer.rv.chat qwenputer-rv \
  --firmware tests/data/rv/calc-bridge.elf \
  --ask 'Use calc: what is 123456789 * 987654321?' --check

uv run python -m chipformer.rv.chat qwenputer-rv \
  --ask 'Use calc: what is gcd(1234567890,9876543210)?' --check

uv run python -m chipformer.rv.chat qwenputer-rv \
  --ask 'Use calc: what is (2**64 - 1) // 12345? Use integer floor division.' --check
```

For a smaller, standalone compiled CPU checkpoint (without Qwen's language layers):

```bash
uv run python -m chipformer.rv.tinycheck --device cuda \
  --progs signext,call,memcpy,c-crc32p,c-strops
```

Full-model results on RTX 3090 (generation time excludes model loading and replay):

| Prompt | Answer | Generation | CPU trace tokens | Replay |
| --- | --- | --- | --- | --- |
| `123456789 * 987654321` | `121932631112635269` | 12.1 s | 2,430 | Passed, 0 wrong predictions |
| `gcd(1234567890,9876543210)` | `90` | 12.9 s | 3,351 | Passed, 0 wrong predictions |
| `(2**64 - 1) // 12345` | `1494268454735484` | 17.8 s | 5,476 | Passed, 0 wrong predictions |

All three runs had zero head mismatches and no lock error. The `calc` responses came from the
compiled CPU path; the reference machines checked the resulting trace.

The standalone CPU also passed `signext` (44 tokens), `call` (474), `memcpy` (159), and
compiled C `c-crc32p` (6,282, output `crc32=414fa339`) and `c-strops` (1,073). Its generated sequences equaled
the reference token for token, and circuit residuals equaled the IR evaluator.

Optional `flash-linear-attention` and `causal-conv1d` kernels were not installed. Transformers
used its PyTorch fallback implementations, so these are correctness demos, not optimized benchmarks.
The upstream tokenizer also emitted a regex warning; no tokenizer files were changed.

Local raw logs: `local-results/2026-11-04/` (ignored by Git).
Reference firmware test suite in the installed environment: **130 passed, 4 skipped**.
