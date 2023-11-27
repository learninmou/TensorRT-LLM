# Yi

This document shows how to build and run a model [Yi](https://github.com/01-ai/Yi) (including [Yi-6B](https://huggingface.co/01-ai/Yi-6B) / [Yi-34B](https://huggingface.co/01-ai/Yi-34B)) in TensorRT-LLM, both tensor parallelism acceleration and pipeline parallelism acceleration are supported. Please note that Yi models are compatible with Llama. You can also follow the examples provided under the [examples/llama](../llama/) directory.

## Overview

The TensorRT-LLM Yi example code is located in [`examples/yi`](./). 
Main files in that folder::

 * [`build.py`](./build.py) to build the [TensorRT](https://developer.nvidia.com/tensorrt) engine(s) needed to run the Yi model
 * [`run.py`](./run.py) to run the inference on an input text or an input file
 * [`summarize.py`](./summarize.py) to summarize the articles in the [cnn_dailymail](https://huggingface.co/datasets/cnn_dailymail) dataset using the model.


## Support Matrix
  * BF16
  * FP16
  * FP32
  * INT8 & INT4 Weight-Only
  * SmoothQuant
  * Groupwise quantization (AWQ/GPTQ)
  * FP8 KV CACHE
  * INT8 KV CACHE (+ AWQ/per-channel weight-only)
  * Tensor Parallel

## Usage

The TensorRT-LLM Yi example code locates at [examples/yi](./). It takes Huggingface(HF) model weights as input, and builds the corresponding TensorRT engines. The number of TensorRT engines depends on the number of GPUs used to run inference.

### Build TensorRT engine(s)

First you need to install dependencies :
```bash
pip install -r requirements.txt
```
To specify the HF Yi checkpoint path, you can download it from some online repositories, like Huggingface([Yi-6B](https://huggingface.co/01-ai/Yi-6B) / [Yi-34B](https://huggingface.co/01-ai/Yi-34B)) or Modelscope([Yi-6B](https://modelscope.cn/models/01ai/Yi-6B/summary) / [Yi-34B](https://modelscope.cn/models/01ai/Yi-34B/summary))

TensorRT-LLM Yi builds TensorRT engine(s) from HF checkpoint. If no checkpoint directory is specified, TensorRT-LLM will build engine(s) with dummy weights.

Normally `build.py` only requires single GPU, but if you've already got all the GPUs needed while inferencing, you could enable parallelly building to make the engine building process faster by adding `--parallel_build` argument. Please note that currently `parallel_build` feature only supports single node.

Here're some examples :

```bash
# Enable the special TensorRT-LLM GPT Attention plugin (--use_gpt_attention_plugin) to increase runtime performance.
# Try use_gemm_plugin to prevent accuracy issue.

# Build Yi-6B model using a single GPU with bfloat16 datatype
python build.py \
    --model_dir 01-ai/Yi-6B \
    --dtype bfloat16 \
    --log_level info \
    --output_dir trtllm_models/yi-6b_tp1_pp1 \
    --remove_input_padding \
    --use_gpt_attention_plugin bfloat16 \
    --use_gemm_plugin bfloat16 \
    --enable_context_fmha \
    --use_fused_mlp

# Build the Yi-34B model using a 4 GPU devices with tensor parallelism
python build.py \
    --world_size 4 \
    --tp_size 4 \
    --pp_size 1 \
    --model_dir 01-ai/Yi-34B \
    --dtype bfloat16 \
    --log_level info \
    --output_dir trtllm_models/yi-34b_tp4_pp1 \
    --max_input_len 1024 \
    --max_output_len 512 \
    --remove_input_padding \
    --use_gpt_attention_plugin bfloat16 \
    --use_gemm_plugin bfloat16 \
    --enable_context_fmha \
    --parallel_build \
    --use_parallel_embedding \
    --use_fused_mlp
```
### Run

To run a TensorRT-LLM Yi model using the engines generated by build.py :

```bash
# run Yi-6B with single GPU device
python run.py \
    --max_output_len 100 \
    --log_level info \
    --engine_dir trtllm_models/yi-6b_tp1_pp1 \
    --tokenizer_dir 01-ai/Yi-6B \
    --input_text "1,2,3,4,5,6,7,"

# run Yi-34B with 4 GPU devices (tensor_parallel=4)
mpirun -n 4 --allow-run-as-root \
    python run.py \
        --max_output_len 100 \
        --log_level info \
        --engine_dir trtllm_models/yi-34b_tp4_pp1 \
        --tokenizer_dir 01-ai/Yi-34B \
        --input_text "1,2,3,4,5,6,7,"
```


#### INT8 KV cache
INT8 KV cache could be enabled to reduce memory footprint. It will bring more performance gains when batch size gets larger.

You can get the INT8 scale of KV cache through [`hf_yi_convert.py`](./hf_yi_convert.py), which features a
`--calibrate-kv-cache, -kv` option. Setting `-kv` will calibrate the model,
and then export the scaling factors needed for INT8 KV cache inference.


Example:

```bash
python3 hf_yi_convert.py -i 01-ai/Yi-6B -o yi-6b/int8_kv_cache/ --calibrate-kv-cache -t fp16
```

[`build.py`](./build.py) add new options for the support of INT8 KV cache.

`--int8_kv_cache` is the command-line option to enable INT8 KV cache, and `--ft_model_dir` should contain the directory where the INT8 KV cache scales lie in.

**INT8 KV cache + per-channel weight-only quantization**

INT8 KV cache could be combined with per-channel weight-only quantization, as follows:

Examples of INT8 weight-only quantization + INT8 KV cache

```bash
# Build model with both INT8 weight-only and INT8 KV cache enabled
python build.py --ft_model_dir=yi-6b/int8_kv_cache/1-gpu/ \
                --dtype float16 \
                --use_gpt_attention_plugin float16 \
                --use_gemm_plugin float16 \
                --output_dir trtllm_models/yi-6b/int8_kv_cache_weight_only/1-gpu \
                --int8_kv_cache \
                --use_weight_only
```


Test with `summarize.py`:

```bash
python summarize.py --test_trt_llm \
                    --hf_model_location 01-ai/Yi-6B \
                    --data_type fp16 \
                    --engine_dir trtllm_models/yi-6b/int8_kv_cache_weight_only/1-gpu \
                    --test_hf
```

**INT8 KV cache + AWQ**

In addition, you can enable INT8 KV cache together with AWQ (per-group INT4 weight-only quantization)like the following command.

**NOTE**: AWQ checkpoint is passed through `--model_dir`, and the INT8 scales of KV cache is through `--ft_model_dir`.

```bash
python build.py --model_dir 01-ai/Yi-6B \
                --quant_ckpt_path ./yi-6b-4bit-gs128-awq.pt \
                --dtype float16 \
                --remove_input_padding \
                --use_gpt_attention_plugin float16 \
                --enable_context_fmha \
                --use_gemm_plugin float16 \
                --use_weight_only \
                --weight_only_precision int4_awq \
                --per_group \
                --output_dir trtllm_models/yi-6b/int8_kv_cache_int4_AWQ/1-gpu/
                --int8_kv_cache \ # Turn on INT8 KV cache
                --ft_model_dir yi-6b/int8_kv_cache/1-gpu/ # Directory to look for INT8 scale of KV cache
```

Test with `summarize.py`:

```bash
python summarize.py --test_trt_llm \
                    --hf_model_location 01-ai/Yi-6B \
                    --data_type fp16 \
                    --engine_dir trtllm_models/yi-6b/int8_kv_cache_int4_AWQ/1-gpu \
                    --test_hf
```

#### SmoothQuant

Unlike the FP16 build where the HF weights are processed and loaded into the TensorRT-LLM directly, the SmoothQuant needs to load INT8 weights which should be pre-processed before building an engine.

Example:
```bash
python3 hf_yi_convert.py -i 01-ai/Yi-6B -o yi-6b/smoothquant/sq0.8/ -sq 0.8 --tensor-parallelism 1 --storage-type fp16
```

[`build.py`](./build.py) add new options for the support of INT8 inference of SmoothQuant models.

`--use_smooth_quant` is the starting point of INT8 inference. By default, it
will run the model in the _per-tensor_ mode.

Then, you can add any combination of `--per-token` and `--per-channel` to get the corresponding behaviors.

Examples of build invocations:

```bash
# Build model for SmoothQuant in the _per_tensor_ mode.
python3 build.py --ft_model_dir=yi-6b/smoothquant/sq0.8/1-gpu/ \
                 --use_smooth_quant

# Build model for SmoothQuant in the _per_token_ + _per_channel_ mode
python3 build.py --ft_model_dir=yi-6b/smoothquant/sq0.8/1-gpu/ \
                 --use_smooth_quant \
                 --per_token \
                 --per_channel
```

Note we use `--ft_model_dir` instead of `--model_dir` and `--meta_ckpt_dir` since SmoothQuant model needs INT8 weights and various scales from the binary files.

#### FP8 Post-Training Quantization

The examples below uses the NVIDIA AMMO (AlgorithMic Model Optimization) toolkit for the model quantization process.

First make sure AMMO toolkit is installed (see [examples/quantization/README.md](/examples/quantization/README.md#preparation))

After successfully running the script, the output should be in .npz format, e.g. `quantized_fp8/yi_tp_1_rank0.npz`,
where FP8 scaling factors are stored.

```bash
# Quantize HF Yi-34B into FP8 and export a single-rank checkpoint
python quantize.py --model_dir 01-ai/Yi-34B \
                   --dtype float16 \
                   --qformat fp8 \
                   --export_path ./quantized_fp8 \
                   --calib_size 512 \

# Build Yi-34B TP=4 using original HF checkpoint + PTQ scaling factors from the single-rank checkpoint
python build.py --model_dir 01-ai/Yi-34B \
                --quantized_fp8_model_path ./quantized_fp8/llama_tp1_rank0.npz \
                --dtype float16 \
                --use_gpt_attention_plugin float16 \
                --use_gemm_plugin float16 \
                --output_dir trtllm_models/yi-34b/fp8/2-gpu/ \
                --remove_input_padding \
                --enable_fp8 \
                --fp8_kv_cache \
                --world_size 4 \
                --tp_size 4
```

#### Groupwise quantization (AWQ/GPTQ)
One can enable AWQ/GPTQ INT4 weight only quantization with these options when building engine with `build.py`:

- `--use_weight_only` enables weight only GEMMs in the network.
- `--per_group` enable groupwise weight only quantization, for GPT-J example, we support AWQ with the group size default as 128.
- `--weight_only_precision` should specify the weight only quantization format. Supported formats are `int4_awq` or `int4_gptq`.
- `--quant_ckpt_path` passes the quantized checkpoint to build the engine.

AWQ/GPTQ examples below involves 2 steps:
1. Weight quantization
2. Build TRT-LLM engine

##### AWQ
1. Weight quantization:

    NVIDIA AMMO toolkit is used for AWQ weight quantization. Please see [examples/quantization/README.md](/examples/quantization/README.md#preparation) for AMMO installation instructions.

    ```bash
    # Quantize HF Yi-6B checkpoint into INT4 AWQ format
    python quantize.py --model_dir 01-ai/Yi-6B \
                    --dtype float16 \
                    --qformat int4_awq \
                    --export_path ./yi-6b-4bit-gs128-awq.pt \
                    --calib_size 32
    ```
    The quantized model checkpoint is saved to path `./yi-6b-4bit-gs128-awq.pt` for future TRT-LLM engine build.

2. Build TRT-LLM engine:

    ```bash
    python build.py --model_dir 01-ai/Yi-6B \
                    --quant_ckpt_path ./yi-6b-4bit-gs128-awq.pt \
                    --dtype float16 \
                    --remove_input_padding \
                    --use_gpt_attention_plugin float16 \
                    --enable_context_fmha \
                    --use_gemm_plugin float16 \
                    --use_weight_only \
                    --weight_only_precision int4_awq \
                    --per_group \
                    --output_dir trtllm_models/yi-6b/int4_AWQ/1-gpu/
    ```

##### GPTQ
To run the GPTQ Yi model example, the following steps are required:

1. Weight quantization:

    Quantized weights for GPTQ are generated using [GPTQ-for-LLaMa](https://github.com/qwopqwop200/GPTQ-for-LLaMa.git) as follow:

    ```bash
    git clone https://github.com/qwopqwop200/GPTQ-for-LLaMa.git
    cd GPTQ-for-LLaMa
    pip install -r requirements.txt

    # Quantize weights into INT4 and save as safetensors
    # Quantized weight with parameter "--act-order" is not supported in TRT-LLM
    python llama.py 01-ai/Yi-6B c4 --wbits 4 --true-sequential --groupsize 128 --save_safetensors ./yi-6b-4bit-gs128.safetensors
    ```

    Let us build the TRT-LLM engine with the saved `./yi-6b-4bit-gs128.safetensors`.

2. Build TRT-LLM engine:

    ```bash
    # Build the Yi-6B model using INT4 GPTQ quantization.
    # Compressed checkpoint safetensors are generated separately from GPTQ.
    python build.py --model_dir 01-ai/Yi-6B \
                    --quant_ckpt_path ./yi-6b-4bit-gs128.safetensors \
                    --dtype float16 \
                    --remove_input_padding \
                    --use_gpt_attention_plugin float16 \
                    --enable_context_fmha \
                    --use_gemm_plugin float16 \
                    --use_weight_only \
                    --weight_only_precision int4_gptq \
                    --per_group \
                    --world_size 1 \
                    --tp_size 1 \
                    --output_dir trtllm_models/yi-6b/int4_GPTQ/2-gpu/
    ```