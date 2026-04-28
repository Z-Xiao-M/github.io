本次部署大模型我们这里使用的是[llama.cpp](https://github.com/ggml-org/llama.cpp)，待部署的模型是Qwen3-0.6B（其实很小）

### [llama.cpp](https://github.com/ggml-org/llama.cpp)的源码编译安装

```bash
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build
cmake --build build --config Release # 可以加并行编译 如：-j4
```

编译完后的可执行文件位于llama.cpp/build/bin 我们一会需要用到其中的部分可执行程序

```bash
postgres@zxm-VMware-Virtual-Platform:~/llama.cpp/build/bin$ ls
libggml-base.so      llama-cli                      llama-gemma3-cli  llama-lookahead      llama-passkey          llama-server              test-alloc        test-chat-template           test-log                test-rope
libggml-cpu.so       llama-convert-llama2c-to-ggml  llama-gen-docs    llama-lookup         llama-perplexity       llama-simple              test-arg-parser   test-gbnf-validator          test-model-load-cancel  test-sampling
libggml.so           llama-cvector-generator        llama-gguf        llama-lookup-create  llama-q8dot            llama-simple-chat         test-autorelease  test-gguf                    test-mtmd-c-api         test-thread-safety
libllama.so          llama-diffusion-cli            llama-gguf-hash   llama-lookup-merge   llama-quantize         llama-speculative         test-backend-ops  test-grammar-integration     test-opt                test-tokenizer-0
libmtmd.so           llama-embedding                llama-gguf-split  llama-lookup-stats   llama-qwen2vl-cli      llama-speculative-simple  test-barrier      test-grammar-parser          test-quantize-fns       test-tokenizer-1-bpe
llama-batched        llama-eval-callback            llama-imatrix     llama-minicpmv-cli   llama-retrieval        llama-tokenize            test-c            test-json-partial            test-quantize-perf      test-tokenizer-1-spm
llama-batched-bench  llama-export-lora              llama-llava-cli   llama-mtmd-cli       llama-run              llama-tts                 test-chat         test-json-schema-to-grammar  test-quantize-stats
llama-bench          llama-finetune                 llama-logits      llama-parallel       llama-save-load-state  llama-vdot                test-chat-parser  test-llama-grammar           test-regex-partial
```

### &#x20;下载Qwen3-0.6B

模型我们可以前往huggingface下载，这里给出下载链接：<https://huggingface.co/Qwen/Qwen3-0.6B-GGUF?show_file_info=Qwen3-0.6B-Q8_0.gguf>

<img width="2552" height="1284" alt="Image" src="https://github.com/user-attachments/assets/e9a7eeb4-a918-4019-b38a-ecc1f0aeb977" />

然后点击Download即可，文件大小639MB

### &#x20;启动模型

#### 控制台启动

[`llama-cli`](https://github.com/ggml-org/llama.cpp/blob/master/tools/main)是一个控制台程序，可用于与 LLM 聊天。这里我们使用本地的模型，所以使用-m执行本地的模型文件

```bash
# 进入可执行文件路径
cd llama.cpp/build/bin
# 使用本地的模型，所以使用-m执行本地的模型文件 
# 我将下载下来的文件放在了~/model/Qwen3-0.6B-Q8_0.gguf
# 请按需更改
./llama-cli -m ~/model/Qwen3-0.6B-Q8_0.gguf
```

运行结果

```bash
./llama-cli -m ~/model/Qwen3-0.6B-Q8_0.gguf
# > 你是谁
# <think>
# 好的，用户问“你是谁”。作为一个AI助手，我需要明确回答。首先，我应该礼貌地确认自己的身份，然后简要说明我的功能和用途。同时，也要保持友好和专业的形象，让用户感到被理解和支持。在回答中，可以提到帮助用户解决问题和提供信息，这样既符合AI助手的身份，又能满足用户的需求。最后，确保语气自然，避免使用过于机械或生硬的表达。
# </think>
# 
# 我是您的AI助手，很高兴为您服务！如果您有任何问题或需要帮助，请随时告诉我，我会尽力为您提供支持。有什么我可以帮您解决的事情吗？😊
# 
# > 1+1等于多少
# <think>
# 嗯，用户问的是“1+1等于多少”。这是一个很基础的数学问题，但作为一个AI助手，我需要保持专业和礼貌。首先，我要确认用户是否理解这个问题，或者是否需要更详细的解释。1加1的结果是2，这是数学的基本运算。不过，用户可能是在测试我的反应，或者想了解更多背景信息。在这种情况下，直接给出答案是合适的，但也要让用户知道这是基础数学，可能他们需要进一步的数学知识。同时，保持友好和耐心，让用户感到被理解和支持。另外，检查是否有其他潜在的意图，比如是否在进行某种计算，或者是否有其他相关问题需要回答。但当前问题非常直接，所以保持简洁和明确的回应是最佳选择。最后，确保回答符合AI助手的身份，同时提供帮助的意愿。
# </think>
# 
# 1加1的结果是2。如果您有其他问题或需要帮助，随时告诉我！😊

> 
```

#### 网页显示

[llama-server](https://github.com/ggml-org/llama.cpp/tree/master/tools/server) 是一个简单的 HTTP 服务器，包括一组 LLM REST API 和一个简单的 Web 前端，用于使用 llama.cpp 与 LLM 进行交互。

```bash
./llama-server -m ~/model/Qwen3-0.6B-Q8_0.gguf
```

默认通过<http://127.0.0.1:8080/>访问

<img width="1805" height="920" alt="Image" src="https://github.com/user-attachments/assets/49982896-82db-4ef5-8202-697fa6175cdb" />

尝试问一些基础问题

<img width="2125" height="1151" alt="Image" src="https://github.com/user-attachments/assets/eac9ae8a-99c5-4a16-8e08-3a7fcdb4c907" />

### 感想
如果资源足够的话，还是选用更多参数的模型更好，0.6B实在是不够看的。