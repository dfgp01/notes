
以下是我的设备信息
设备：macbook-air-m4
芯片：Apple M4
OS：sequoia 15.6
配置：16+256
显存：系统没找到显卡信息，可能是集成显卡

```code
目前已从git上安装了 comfyUI，系统也装好了brew、python3.11；已建好了venv，并下载了相关的package；已有sd_v1-5.ckpt；也可以成功执行python来启动comfyUI
现有疑问：目前设备可能是集显，即不具备顺利文生图的条件，需要做何更改或调整？另外有些回答会提及到不能用CUDA，而用MPS之类的一些建议
```



---
# 以下是旧案

## 常见后续需求
 
自动安装 LoRA / ControlNet 预处理器，在脚本尾部追加：
```bash
# 示例：安装 controlnet-aux 预处理节点
cd "$COMFY/custom_nodes"
git clone --depth 1 https://github.com/Fannovel16/comfyui_controlnet_aux.git
cd comfyui_controlnet_aux
pip install -r requirements.txt
```

升级 PyTorch 到 nightly（支持新 MPS 算子）
```bash
pip install --upgrade torch torchvision torchaudio --index-url https://download.pytorch.org/whl/nightly/cpu
```
## 其他疑问

学会在 https://hf-mirror.com/ 这些网站上找到模型，有些大有些小，下载完放在 /work/llm-devlab/models/checkpoints/