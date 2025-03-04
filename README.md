# FlashAttention
This repository provides the official implementation of FlashAttention from the
following paper.

**FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness**  
Tri Dao, Daniel Y. Fu, Stefano Ermon, Atri Rudra, Christopher Ré  
Paper: https://arxiv.org/abs/2205.14135  
IEEE Spectrum [article](https://spectrum.ieee.org/mlperf-rankings-2022) about our submission to the MLPerf 2.0 benchmark using FlashAttention.

### Figure 1
![FlashAttention](assets/flashattn_banner.jpg)

**Left:**
- FlashAttention uses tiling to prevent materialization of the large $N \times N$ attention matrix (dotted box) on (relatively) slow GPU HBM.
- In the outer loops (red arrows), FlashAttention loops through blocks of the **K** and **V** matrices and loads them to fast on-chip SRAM. 
- In each block, FlashAttention loops over blocks of **Q** matrix (blue arrows), loading them to SRAM, and writing the output of the attention computation back to HBM.


**Right:**
- Speedup over the PyTorch implementation of attention on GPT-2.
- FlashAttention does not read and write the large $N \times N$ attention matrix to HBM, resulting in an $7.6\times$ speedup on the attention computation.


## Usage

We've been very happy to see FlashAttention being widely adopted in such a short
time after its release. This [page](https://github.com/HazyResearch/flash-attention/blob/main/usage.md)
contains a partial list of places where FlashAttention is being used.

## Full model code and training script

We have released the full GPT model
[implementation](https://github.com/HazyResearch/flash-attention/blob/main/flash_attn/models/gpt.py).
We also provide optimized implementations of other layers (e.g., MLP, LayerNorm,
cross-entropy loss, rotary embedding). Overall this speeds up training by 3-5x
compared to the baseline implementation from Huggingface, reaching up to 189
TFLOPs/sec per A100, equivalent to 60.6\% model FLOPs utilization (we don't need
any activation checkpointing). 

We also include a training
[script](https://github.com/HazyResearch/flash-attention/tree/main/training) to
train GPT2 on Openwebtext and GPT3 on The Pile.

## Triton implementation of FlashAttention

Phil Tillet (OpenAI) has an experimental implementation of FlashAttention in Triton:
https://github.com/openai/triton/blob/master/python/tutorials/06-fused-attention.py  

As Triton is a higher-level language than CUDA, it might be easier to understand
and experiment with. The notations in the Triton implementation are also closer
to what's used in our paper.


## Installation and features

Requirements:
- CUDA 11.4 and above.
- PyTorch 1.12 and above.

To install:
```sh
pip install flash-attn
```

Alternatively you can compile from source:
```
python setup.py install
```

Interface: `src/flash_attention.py`

To run the benchmark against PyTorch standard attention: 
```
PYTHONPATH=$PWD python benchmarks/benchmark_flash_attention.py
```

FlashAttention currently supports:
1. Turing, Ampere, Ada, or Hopper GPUs (e.g., H100, A100, RTX 3090, T4, RTX 2080).
2. fp16 and bf16 (bf16 requires Ampere, Ada, or Hopper GPUs).
3. Head dimensions that are multiples of 8, up to 128 (e.g., 8, 16, 24, ...,
   128). Head dim > 64 backward requires A100 or H100.

Our tentative roadmap:
1. ~~[Jun 2022] Make package pip-installable~~[Done, thanks to lucidrains].
2. ~~[Jun 2022] Support SM86 GPUs (e.g., RTX 3080, 3090)~~[Done].
3. ~~[Jun 2022] Support SM75 GPUs (e.g. T4)~~[Done].
4. ~~[Jun 2022] Support bf16~~[Done].
5. ~~[Jul 2022] Implement cross-attention~~[Done].
6. ~~[Jul 2022] Support head dimension 128~~[Done].
7. ~~[Aug 2022] Fuse rotary embedding~~[Done].
8. ~~[Mar 2023] Support SM90 GPUs (H100)~~[Done].
9. [Apr 2023] Refactor to use Cutlass 3.x.
10. [May 2023] Support attention bias (e.g. ALiBi, relative positional encoding).
11. [Jun 2023] Support SM70 GPUs (V100).
12. [Jun 2023] Support fp8 (H100).


## How to use FlashAttention

Here's a simple example:
```python
import torch
from flash_attn.flash_attention import FlashMHA

# Replace this with your correct GPU device
device = "cuda:0"

# Create attention layer. This is similar to torch.nn.MultiheadAttention,
# and it includes the input and output linear layers
flash_mha = FlashMHA(
    embed_dim=128, # total channels (= num_heads * head_dim)
    num_heads=8, # number of heads
    device=device,
    dtype=torch.float16,
)

# Run forward pass with dummy data
x = torch.randn(
    (64, 256, 128), # (batch, seqlen, embed_dim)
    device=device,
    dtype=torch.float16
)

output = flash_mha(x)[0]
```

Alternatively, you can import the inner attention layer only (so that the input
and output linear layers are not included):
```python
from flash_attn.flash_attention import FlashAttention

# Create the nn.Module
flash_attention = FlashAttention()
```

Or, if you need more fine-grained control, you can import one of the lower-level
functions (this is more similar to the `torch.nn.functional` style):
```python
from flash_attn.flash_attn_interface import flash_attn_unpadded_func

# or

from flash_attn.flash_attn_interface import flash_attn_unpadded_qkvpacked_split_func

# etc.
```

There are also separate Python files with various FlashAttention extensions:
```python
# Import the triton implementation (torch.nn.functional version only)
from flash_attn.flash_attn_triton import flash_attn_func

# Import block sparse attention (nn.Module version)
from flash_attn.flash_blocksparse_attention import FlashBlocksparseMHA, FlashBlocksparseAttention

# Import block sparse attention (torch.nn.functional version)
from flash_attn.flash_blocksparse_attn_interface import flash_blocksparse_attn_func
```

## Speedup and Memory Savings

We present expected speedup (combined forward + backward pass) and memory savings from using FlashAttention against PyTorch standard attention, depending on sequence length, on different GPUs (speedup depends on memory bandwidth - we see more speedup on slower GPU memory).

We currently have benchmarks for these GPUs:
* [A100](#a100)
* [RTX 3090](#rtx-3090)
* [T4](#t4)

### A100

We display FlashAttention speedup using these parameters (similar to BERT-base):
* Batch size 8
* Head dimension 64
* 12 attention heads

Our graphs show sequence lengths between 128 and 4096 (when standard attention runs out of memory on an A100), but FlashAttention can scale up to sequence length 64K.

#### Speedup

![FlashAttention speedup](assets/flashattn_speedup.jpg)

We generally see 2-4X speedup at sequence lengths between 128 and 4K, and we see more speedup when using dropout and masking, since we fuse the kernels.
At sequence lengths that are popular with language models like 512 and 1K, we see speedups up to 4X when using dropout and masking.

#### Memory

![FlashAttention memory](assets/flashattn_memory.jpg)

We show memory savings in this graph (note that memory footprint is the same no matter if you use dropout or masking).
Memory savings are proportional to sequence length -- since standard attention has memory quadratic in sequence length, whereas FlashAttention has memory linear in sequence length.
We see 10X memory savings at sequence length 2K, and 20X at 4K.
As a result, FlashAttention can scale to much longer sequence lengths.

#### Head Dimension 128

![FlashAttention speedup, head dimension 128](assets/flashattn_speedup_a100_d128.jpg)

We show speedup with head dimension 128.
Here we show batch size 16 with 12 heads.
Speedup is less than with the smaller head sizes, since we have to make the block size smaller in the tiling.
But speedup is still significant, especially with a causal mask.

### RTX 3090

For the RTX 3090, we use batch size 12 with 12 attention heads.
Memory savings are the same as on an A100, so we'll only show speedup here.

![FlashAttention speedup GTX 3090](assets/flashattn_speedup_3090.jpg)

We see slightly higher speedups (between 2.5-4.5x) on the GTX 3090, since memory bandwidth on the GDDR6X is lower than A100 HBM (~900 GB/s vs. ~1.5 TB/s).

### T4

We again use batch size 12 with 12 attention heads.

![Flashattention speedup T4](assets/flashattn_speedup_t4.jpg)

T4 SRAM is smaller than the newer GPUs (64 KB), so we see less speedup (we need to make the block sizes smaller, so we end up doing more R/W).
This matches the IO complexity analysis from section 3.2 of [our paper](https://arxiv.org/abs/2205.14135).

T4 GPUs are commonly used for inference, so we also measure speedup on the forward pass only (note that these are not directly comparable to the graphs above):

![FlashAttention speedup T4 fwd](assets/flashattn_speedup_t4_fwd.jpg)

We see speedups between 2.5x-4.5x on the forward pass.

## Tests
We test that FlashAttention produces the same output and gradient as a reference
implementation, up to some numerical tolerance. In particular, we check that the
maximum numerical error of FlashAttention is at most twice the numerical error
of a baseline implementation in Pytorch (for different head dimensions, input
dtype, sequence length, causal / non-causal).

To run the tests:
```
pytest -q -s tests/test_flash_attn.py
```
## When you encounter issues

This alpha release of FlashAttention contains code written for a research
project to validate ideas on speeding up attention. 
We have tested it on several models (BERT, GPT2, ViT). 
However, there might still be bugs in the implementation that we hope to iron
out in the next few months.

If you encounter any of these bugs, please open a respective GitHub Issue!

## Acknowledgments
Our implementation uses Apex's
[FMHA](https://github.com/NVIDIA/apex/tree/master/apex/contrib/csrc/fmha) code
as a starting point.

We thank [Young-Jun Ko](https://yjk21.github.io/) for the in-depth explanation of his FMHA implementation
and for his thoughtful answers to our questions about CUDA.

## Citation
If you use this codebase, or otherwise found our work valuable, please cite:
```
@inproceedings{dao2022flashattention,
  title={Flash{A}ttention: Fast and Memory-Efficient Exact Attention with {IO}-Awareness},
  author={Dao, Tri and Fu, Daniel Y. and Ermon, Stefano and Rudra, Atri and R{\'e}, Christopher},
  booktitle={Advances in Neural Information Processing Systems},
  year={2022}
}
```
