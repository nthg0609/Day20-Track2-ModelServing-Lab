# Bonus — Thread sweep

Model: `tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf`  ·  GPU layers: `0`

| threads | tg128 (tok/s) |
|---:|---:|
| 1 | 20.2 |
| 2 | 29.1 |
| 4 | 43.1 |
| 8 | 36.8 |
| 16 | 30.2 |
| 32 | 29.9 |

**Best**: `-t 4` at 43.1 tok/s.

Look at the curve. If it peaks around your **physical** core count and drops as you go higher, that's the memory-bandwidth ceiling: extra threads fight over the same memory channels and slow each other down.
