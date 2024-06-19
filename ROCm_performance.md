# Overview of the optional performance features uinque to https://github.com/ROCm/vllm
## Multi-GPU torchrun
On ROCm the default multi GPU executor is `torchrun` as opposed to `ray` on NVIDIA  
This can be overridden by the `--worker-use-ray` flag to vllm or its benchmarks  
To utilize torchran parallelism, the run command should be modified from  
`python <command>`  
to  
`torchrun --standalone --nnodes=1 --nproc-per-node=<world-size> <command>`
## Triton attention
The default attention function on ROCm is using triton attention kernel. To fallback to the https://github.com/ROCm/flash-attention implementation set up the following environment symbol:  
`VLLM_USE_FLASH_ATTN_TRITON=False`
## Tunable ops
Pytorch tunable ops are supported.  
Define the following environment symbol: `PYTORCH_TUNABLEOP_ENABLED=1` in order to enable both the runtime tuning and the subsequent use of tuned results. To only use the tuned results without tuning any newly encountered shapes, also define `PYTORCH_TUNABLEOP_TUNING=1`

## Custom PagedAttention

On ROCm, to have better performance, a custom paged attention is available by switching on the env variable: `VLLM_USE_ROCM_CUSTOM_PAGED_ATTN=1`.
Currently, this env variable is enabled by default. To fallback to PagedAttention v2 kernel assign the env variable to 0.
The custom PagedAttention kernel is enabled for dtype: fp16, block-size=16, head-size=128, and max context length <= 16k, with GQA ratio (num_heads//num_kv_heads) between 1 to 16. On all the other cases, we fallback to PagedAttention v2 kernel.

## Fp8 Quantization

To use fp8 quantization, first step is to quantize your model to fp8 format. 

By default, rocm-vllm accepts the quantized weights generated by Quark quantizer. To do this, install quark and run the command:

```
python3 quantize_quark.py --model_dir [llama2 checkpoint folder] \
                          --output_dir output_dir \
                          --quant_scheme w_fp8_a_fp8_o_fp8 \
                          --num_calib_data 128 \
                          --model_export vllm_adopted_safetensors \
                          --no_weight_matrix_merge
```
For more details, please refer to Quark's documentation.

To use ammo, please follow this [instruction](https://github.com/ROCm/vllm/tree/main/examples/fp8/quantizer), and set `VLLM_FP8_USE_AMMO=1`.

Both quantizers generate a safetensor file that contains the quantized weights and the corresponding scaling factors of your model. The safetensor file should be placed under your model folder. Then we can run a model with fp8 quantization using vllm. When creating `vllm.LLM` object, two additional parameters should be added: `quantization="fp8"` and `quantization_param_path={relative path of the safetensors with your model path}`.

## Gemm Tuning for Fp8

To get better performance of fp8 quantization, we will need to tune the gemm with the information of all the shapes used in the execution of the model. 

To obtain all the shapes of gemms during the execution of the model, set the env value `TUNE_FP8=1` and then run the model as usual. We will get the a file called `/tmp/fp8_shapes.csv`.

Next, run gradlib to obtain the best solutions of these shapes:

```
python3 gradlib/gradlib/fp8_gemm_tuner.py --input_file /tmp/fp8_shapes.csv --tuned_file /tmp/tuned_fp8_16.csv
```
where `/tmp/tuned_fp8_16` will be used by our fp8 gemm linear layer.

Now, when running inference with fp8, we are using the tuned gemm for best performance.