## 契机
我花了点时间写了一个用c语言驱动[onnxruntime](https://github.com/microsoft/onnxruntime)，然后加载onnx模型以及推理的一个简单示例（或工具）。由于我对于取名方面实在是没有什么天赋，所以就随便叫[onnx_model](https://github.com/Z-Xiao-M/onnx_model)好了。做这个的原因是在于我翻看了onnxruntime给出的[教程](https://onnxruntime.ai/docs/tutorials/)还有它实际的项目，都没有一个很好的示例。最后我在discussions中翻到了一个和我想法很契合的帖子，帖子名为`Trying to load an onnx file in C`，看到了下面的对话。

<img width="935" height="639" alt="Image" src="https://github.com/user-attachments/assets/6c61e110-b12a-4fd4-9f38-55111b014cee" />
然后我点开了人家给出的链接，也就是onnxrt c example，点开后的页面是这样的

<img width="962" height="226" alt="Image" src="https://github.com/user-attachments/assets/396a6b86-de2d-416b-9f30-2511c06056a7" />

最后悬着的心终于死了，所以决定自己写一个。

## 过程
这方面其实倒是没啥好写的了，其实就是看看文档，然后这里瞅瞅那里瞅瞅，虽然没找到合适的c语言示例，但是好在别的语言封装的onnxruntime还不少，尤其是c++，好在以前学过一点c++，虽然现在其实忘得差不多了，但是勉勉强强还是能够猜到人家那样子写代码的意图的，再借助一点大模型之力，就这样缝缝补补给水了出来（AI大法好呀~）。

## 简单介绍
### 编译安装
onnx_model依赖onnxruntime和cjson，onnxruntime可以通过`download-onnxruntime-linux.sh`或者自行安装，cjson也大差不差。要进行编译安装的话，在相关依赖都已经完事之后，需要配置一下环境变量`ONNXRUNTIME_HOME`，然后使用`make`命令即可。
```bash
git clone https://github.com/Z-Xiao-M/onnx_model.git
cd onnx_model
make
```
使用的话，需要提供代价在的onnx模型和实际的张量数据（需要json格式），一个简单的示例如下
```bash
(onnx-env) postgres@zxm-VMware-Virtual-Platform:~/onnx_model$ ./onnx_model ./model/sample/1/model.onnx ./model/sample/1/input.json
============= inputs =============
inputs: "{"x": "float32[-1, 1]"}"
============= outputs =============
outputs: "{"y": "float32[-1, 1]"}"
============= result =============
result: [4.9851140975952148437500, 7.9896388053894042968750, 10.9941635131835937500000, 13.9986886978149414062500]
```
整体的目录如下，非常的简单，整个项目包含两个onnx模型
- model.onnx，这是一个线性回归模型，其实就是`y ≈ 3x + 2 + noise`, 由inear_regression_onnx_export.py生成，可通过阅读python代码了解详情。
- [det.onnx](https://github.com/Z-Xiao-M/OnnxOCR_PG_ONNX/tree/main/onnxocr/models/ppocrv5/det/det.onnx)，这是一个ocr的detection模型，值得注意的是使用`./onnx_model ./model/sample/2/det.onnx ./model/sample/2/input.json`会输出海量数据。
```bash
(onnx-env) postgres@zxm-VMware-Virtual-Platform:~/onnx_model$ tree
.
├── download-onnxruntime-linux.sh
├── include
│   └── parse_json.h
├── main.c
├── Makefile
├── model
│   └── sample
│       ├── 1
│       │   ├── input.json
│       │   ├── linear_regression_onnx_export.py
│       │   ├── model.onnx
│       │   └── model.onnx.data
│       └── 2
│           ├── det.onnx
│           └── input.json
├── parse_json.c
└── README.md
```
## 备忘
当前版本仅配置了使用cpu