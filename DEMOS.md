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
```

For a smaller, standalone compiled CPU checkpoint (without Qwen's language layers):

```bash
uv run python -m chipformer.rv.tinycheck --device cuda \
  --progs signext,call,memcpy,c-crc32p,c-strops
```

Model inference results will be recorded after execution; the reference results above
must not be interpreted as full-model inference results.
