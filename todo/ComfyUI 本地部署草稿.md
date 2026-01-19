## ComfyUI 本地部署需求指令

## 环境信息
- 机器：MacBook Air m4 16g+256g 
- 操作系统：macOS sequoia 15.6 (24G84)
- 芯片：Apple M4
- 已安装工具：brew、python3、git
- 网络：可访问 GitHub、Hugging Face

## 部署目标
* 工作目录：
	* 根目录 /Users/hugo/work/llm-devlab/
	* 依赖库 /Users/hugo/work/llm-devlab/lib
	* 模型库 /Users/hugo/work/llm-devlab/models
* 所有下载依赖项目优先访问国内加速镜像网站（如hf-mirror.com、modelscope.cn、 deepseek等）
- 安装 ComfyUI 最新版
- 下载并配置 SD 1.5 模型（推荐写实风格）
- 配置 LoRA 支持
- 启动后可通过浏览器访问 http://127.0.0.1:8188
- 无需 CUDA（使用 MPS 加速）

## 输出要求
请生成一份完整的 shell 脚本（bash），包含：
- 创建虚拟环境
- 克隆 ComfyUI
- 安装依赖（torch、mps 支持）
- 下载模型（自动下载 SD1.5 模型）
- 启动命令
- 目录结构说明

## Kimi给到的脚本（经多次修改）

```bash
#!/usr/bin/env bash

# ComfyUI 全国内镜像部署脚本（macOS M4 专用）
# 工作目录：~/work/llm-devlab
# 强制国内镜像：pip、pytorch、github、hf

# 任何命令返回非0，直接退出
set -e

##############  用户可改区域  ##############

WORK="$HOME/work/llm-devlab"
COMFY="$WORK/ComfyUI"
VENV="$WORK/venv"
MODEL="$WORK/models"

##############  国内镜像常量  ##############

PIP_INDEX="https://pypi.tuna.tsinghua.edu.cn/simple"
PYTORCH_INDEX="https://mirrors.aliyun.com/pytorch-wheels/cpu"
GITHUB_MIRROR="https://github.com"          # 稳定镜像
HF_MIRROR="https://hf-mirror.com"

##########################################

# 1. 目录准备
mkdir -p "$WORK" "$MODEL"/{checkpoints,loras}
cd "$WORK"

# 2. 创建并激活虚拟环境
python3.11 -m venv "$VENV"
source "$VENV/bin/activate"
python --version

# 3. 写入 pip 国内镜像（仅当前venv）
mkdir -p "$VENV/pip.conf"
cat > "$VENV/pip.conf" <<EOF
[global]
index-url = $PIP_INDEX
trusted-host = pypi.tuna.tsinghua.edu.cn
EOF


# 4. 安装 PyTorch（阿里云 CPU 镜像，含 MPS）
pip install --upgrade pip
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu

# 5. 克隆 ComfyUI（国内镜像）
if [[ ! -d "$COMFY" ]]; then
    git clone --depth 1 "$GITHUB_MIRROR/comfyanonymous/ComfyUI.git" "$COMFY"
    
    # 国内 gitee 也有
    # git clone --depth 1 https://gitee.com/mirrors/ComfyUI.git "$COMFY"

else
    git -C "$COMFY" pull
fi

# 6. 安装 ComfyUI 依赖（同样走清华源）
pip install -r "$COMFY/requirements.txt"

# 7. 下载 SD1.5 写实模型（hf-mirror）
CKPT="$MODEL/checkpoints/sd_v1-5.ckpt"

if [[ ! -f "$CKPT" ]]; then
    echo ">>> 从 hf-mirror 下载 SD1.5 模型..."
    pip install -q huggingface_hub
    export HF_ENDPOINT="$HF_MIRROR"
    python3 - <<'PY'
from huggingface_hub import hf_hub_download
hf_hub_download(repo_id="runwayml/stable-diffusion-v1-5",filename="v1-5-pruned-emaonly.ckpt",local_dir_use_symlinks=False,local_dir="/tmp") import shutil, os

shutil.move("/tmp/v1-5-pruned-emaonly.ckpt",os.path.expanduser("~/work/llm-devlab/models/checkpoints/sd_v1-5.ckpt"))

PY

fi

# 8. 软链接模型目录
ln -sfn "$MODEL" "$COMFY/models"

# 9. 启动 ComfyUI
echo ">>> 启动 ComfyUI..."
cd "$COMFY"

# 自动进入 ComfyUI 虚拟环境
python main.py --listen --port 8188

# 后台运行
# nohup python main.py --listen --port 8188 > comfy.log 2>&1 &
```


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

checkpoints,loras,vae,controlnet,clip_vision 这些是什么

comfyUI是纯python项目，核心逻辑、节点系统、模型加载器全部用 PyTorch + Python 写成。没有 Go/Rust/C++ 编译型主程序，入口就是  main.py。

### 为何不用wget而使用python内嵌来下载模型？

官方/社区只把模型托管在 Hugging Face Hub（HfHub）上，而 HfHub 的直链每次 302 跳转到云存储，且 15 分钟后失效。wget 只能拿到一个过期的 302 地址，于是出现「今天脚本还能下，明天就 404」——这正是你前面屡次遇到的情况。

huggingface_hub  CLI/库的优势：
- 自动处理 临时 Bearer token + 302 跳转，永不过期。
- 支持 断点续传、校验 SHA256，大文件下到 99% 断网也不怕。
- HF 镜像站（hf-mirror）同样兼容这套 API，所以脚本里  export HF_ENDPOINT=https://hf-mirror.com  就能走国内加速。
- 一行命令即可指定文件名、版本号，未来换模型不用改脚本。

ModelScope 提供长期 CDN 直链，不会 302 失效：
```bash
# 官方 SD1.5 精简版（固定直链）
wget -c https://www.modelscope.cn/models/AI-ModelScope/stable-diffusion-v1-5/resolve/master/v1-5-pruned-emaonly.safetensors -O sd15_2g.safetensors
```

