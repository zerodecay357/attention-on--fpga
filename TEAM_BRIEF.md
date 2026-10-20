# Team Brief: Attention on FPGA (COL-8085 project)

## 1. What we are making
A hardware **inference accelerator for a tiny transformer** (a mini Vision Transformer), written in **Verilog** and checked in simulation in Vivado 2022.1. It classifies MNIST digits. We do not train from scratch for marks, we just need a trained model whose weights we load into hardware.

The demo: the instructor picks a random image, we show PyTorch's output, then run our Verilog **behavioural simulation** and show the **same numbers**.

## 2. Deadlines
| What | When |
|---|---|
| Files zip (design + testbenches + `.mem` files) | **Thu 29 Oct, 23:59** |
| Demo on GCL | **Wed 4 Nov, 14:00** |
| PPT (15-20 min, in class) | Tue 3 Nov or Fri 6 Nov, 15:30 |
| Report (one PDF, max 5 pages) | **Sun 15 Nov, 23:59** (separate from the files) |
Late files or report = 0 for that part. Our code freeze is **28 Oct**.

## 3. How the model works (read this first)
Image 28x28 -> 16 patches of 7x7 -> each patch becomes a vector of 16 numbers ("token").
One transformer block then does:
1. **Attention:** Q, K, V = three matmuls; scores = Q x K^T; **softmax**; output = weights x V.
2. **FFN:** matmul -> ReLU -> matmul.
3. Average the tokens -> classifier matmul -> argmax = predicted digit.

About 10 matmuls plus one softmax, around 54k multiplications per image. So we build **one reusable matmul engine** and a few small units around it.

**Learn it, in this order (about 3 hours):**
1. [The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/) (Jay Alammar). Picture-based, start here.
2. [Attention in transformers, step-by-step](https://www.3blue1brown.com/lessons/attention) (3Blue1Brown, 26 min). Intuition for Q, K, V. The video is embedded on that lesson page.
3. [Let's build GPT: from scratch, in code, spelled out](https://www.youtube.com/watch?v=kCc8FmEb1nY) (Andrej Karpathy, about 2 h). Code along in PyTorch. He suggests his earlier "makemore" videos first if PyTorch is new to you; the series is at [karpathy.ai/zero-to-hero](https://karpathy.ai/zero-to-hero.html).
4. Section 3 of the Attention paper, then the abstract and Figure 1 of the ViT paper, both in `Research papers/`.

Note: Karpathy's video builds a text (GPT) model, ours is an image model, but the attention block is the same.

## 4. Hardware blocks
| Block | File | Job |
|---|---|---|
| Matmul engine | `rtl/matmul_engine.v` | Parametrized multiply-accumulate array, used for every matmul |
| Requantize + ReLU | `rtl/requant_relu.v` | Rescale int32 results back to int8 |
| Softmax | `rtl/softmax_lut.v` | Integer lookup-table softmax |
| Controller | `rtl/controller.v` | State machine that runs the layers in order |
| Top | `rtl/transformer_top.v` | Wires it together, loads weights from `.mem` files |

**Key decision:** everything is integer (int8 values, int32 accumulators). This is so the hardware output is *bit-exact* with the Python golden model. Floating point would not match.

## 5. Optimizations (each needs a before/after number)
Baseline = one multiplier, sequential. Then: parallel lanes + memory banking, pipelining, one shared engine for all layers, and one **own idea** (SVD on FFN or token pruning). The report must say what is a *decision* and what is an *optimization*, and why.

## 6. Safety net
The same engine also runs an MLP. If the transformer is not matching by **26 Oct**, we ship MLP (and a small CNN) results instead.

## 7. Work split (fill in names)
| Role | Owner | Does |
|---|---|---|
| RTL | | Verilog blocks + testbenches |
| Python / verification | | Train, int8 export, golden model, per-layer diff |
| Synthesis / report / PPT | | Vivado numbers (LUT/FF/BRAM/DSP), tables, report, slides |

## 8. Timeline
| Dates | Work |
|---|---|
| 20-21 Oct | Tools installed, GCL login tested, model trained and weights exported |
| 21-23 Oct | Golden model, matmul engine, MLP matches bit-exactly |
| 23-26 Oct | Softmax, attention, full transformer in simulation, first Vivado synthesis |
| 26-28 Oct | Optimizations, per-module resource numbers, scripts |
| 28-29 Oct | Freeze, zip, submit |
| 30 Oct-3 Nov | Demo rehearsal on GCL, PPT |

## 9. Setup
- **Local (Linux):** `sudo apt install iverilog verilator gtkwave yosys`, `pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu`.
- **GCL (Vivado):** `ssh -X -Y <KerberosID>@gcl1.cse.iitd.ac.in` (UG; `gcl2` for PG), then `source /opt/Xilinx/Vivado/2022.1/settings64.sh`. Collect the password in lecture or office hours. Never commit passwords.
- Write plain standard Verilog so it behaves the same in iverilog and Vivado. Always re-check in Vivado on GCL.
- Use the FPGA part with the **largest LUT/FF** for synthesis.

## 10. Rules to remember
- Inference only; the input is data, never a program.
- Cite the source of every adopted idea; compare your own ideas against the baseline.
- The report lists LUT, FF, BRAM and DSP. For a big design, synthesize per module and extrapolate.
- For the demo, be ready to show the outputs of the **last 2 layers**.

## 11. Papers (in `Research papers/`)
- `Attention_Is_All_You_Need.pdf`: the transformer.
- `ViT_16x16_Words.pdf`: image patches as tokens, our model design.
- `I-BERT_Integer_only.pdf`: integer-only attention, supports our bit-exact approach.
- `Softermax.pdf`: hardware-friendly softmax.
- `2007.08563v1.pdf` (FTRANS): transformer on FPGA.
