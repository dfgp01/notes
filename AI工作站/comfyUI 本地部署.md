
（未验证）先上shell，大致做以下事情：
- 配置工作目录，参数输入，默认是当前PWD
- 从brew开始安装工具集：python, git等
- 设置需要下载的源，优先选国内加速源，如gitee、hf-mirror等
- python创建venv、下载模型
- 下载并启动comfyUI

```bash
#!/usr/bin/env bash
# macOS M 系列 从 0 部署 ComfyUI（Gitee + ModelScope 优先）
# 默认工作目录 = 当前目录；可传入参数覆盖
# 下载顺序：modelscope → hf-mirror → 正源

set -eu

############## 参数与常量 ##############
WORK_DIR="${1:-$PWD}"                 # 默认当前目录
COMFY_DIR="$WORK_DIR/ComfyUI"
MODEL_DIR="$WORK_DIR/models"

PIP_INDEX="https://pypi.tuna.tsinghua.edu.cn/simple"
PYTORCH_INDEX="https://download.pytorch.org/whl/cpu"
HF_MIRROR="https://hf-mirror.com"
MODELSCOPE_REPO="AI-ModelScope/stable-diffusion-v1-5"
MODEL_FILE="v1-5-pruned-emaonly.safetensors"

# 国内镜像优先
GITHUB_MIRROR="https://gitee.com/mirrors/ComfyUI.git"   # Gitee 同步仓
GITHUB_ORIGIN="https://github.com/comfyanonymous/ComfyUI.git"

MINIFORGE_INSTALLER="$HOME/miniforge_installer.sh"
CONDA_EXE="$HOME/miniforge3/bin/conda"
#######################################

# 颜色输出
GREEN='\033[0;32m'; RED='\033[0;31m'; NC='\033[0m'
log() { echo -e "${GREEN}[INFO]${NC} $*"; }
err() { echo -e "${RED}[ERROR]${NC} $*" >&2; }

# 1. 目录准备（当前目录）
log "1. 工作目录：$WORK_DIR"
mkdir -p "$WORK_DIR"/{models/checkpoints,models/loras}
cd "$WORK_DIR"

# 2. 安装 brew（若未装）
if ! command -v brew &>/dev/null; then
    log "2. 安装 Homebrew ..."
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    eval "$(/opt/homebrew/bin/brew shellenv)"
else
    log "2. brew 已存在，跳过"
fi

# 3. 装基础工具
log "3. 安装基础工具 ..."
for tool in python@3.11 git git-lfs cmake wget; do
    brew list "$tool" &>/dev/null || brew install "$tool"
done
git lfs install

# 4. 装 miniforge（若未装）
if [[ ! -f "$CONDA_EXE" ]]; then
    log "4. 安装 miniforge ..."
    curl -L https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-MacOSX-arm64.sh -o "$MINIFORGE_INSTALLER"
    bash "$MINIFORGE_INSTALLER" -b -p "$HOME/miniforge3"
    rm "$MINIFORGE_INSTALLER"
    "$CONDA_EXE" init zsh
    exec zsh
else
    log "4. miniforge 已存在，跳过"
fi

# 5. conda 环境 & PyTorch
source "$HOME/miniforge3/bin/activate"
if ! conda env list | grep -q comfy; then
    log "5. 创建 conda 环境 comfy ..."
    conda create -n comfy python=3.11 -y
fi
conda activate comfy
conda install pytorch torchvision torchaudio -c conda-forge -y

# 6. 国内 pip 镜像（仅当前环境）
mkdir -p "$CONDA_PREFIX/pip.conf"
cat > "$CONDA_PREFIX/pip.conf" <<EOF
[global]
index-url = $PIP_INDEX
trusted-host = pypi.tuna.tsinghua.edu.cn
EOF
export PIP_CONFIG_FILE="$CONDA_PREFIX/pip.conf"

# 7. Python 下载工具
log "6. 安装下载工具 ..."
pip install -U pip
pip install modelscope huggingface_hub

# 8. 克隆 ComfyUI（Gitee 优先）
if [[ ! -d "$COMFY_DIR" ]]; then
    log "7. 克隆 ComfyUI（Gitee优先）..."
    if git clone --depth 1 "$GITHUB_MIRROR" "$COMFY_DIR" 2>/dev/null; then
        log "   使用 Gitee 镜像成功"
    else
        log "   Gitee 失败，回退 GitHub ..."
        git clone --depth 1 "$GITHUB_ORIGIN" "$COMFY_DIR"
    fi
else
    log "7. 更新 ComfyUI ..."
    git -C "$COMFY_DIR" pull || true
fi

# 9. 安装 ComfyUI 依赖
log "8. 安装 ComfyUI 依赖 ..."
pip install -r "$COMFY_DIR/requirements.txt"

# 10. 下载模型：优先 modelscope → hf-mirror → 正源
CKPT="$MODEL_DIR/checkpoints/sd15_2g.safetensors"
if [[ ! -f "$CKPT" ]]; then
    log "9. 下载 SD1.5 模型（modelscope优先）..."
    # ① modelscope 直链（固定 CDN）
    MSC_URL="https://modelscope.cn/api/v1/models/$MODELSCOPE_REPO/repo?Revision=main&FilePath=$MODEL_FILE"
    if wget -c "$MSC_URL" -O "$CKPT.tmp" 2>/dev/null && [[ -s "$CKPT.tmp" ]]; then
        mv "$CKPT.tmp" "$CKPT"
        log "   模型下载完成（modelscope）"
    else
        # ② hf-mirror
        log "   modelscope 失败，尝试 hf-mirror ..."
        export HF_ENDPOINT="$HF_MIRROR"
        if huggingface-cli download runwayml/stable-diffusion-v1-5 \
               "$MODEL_FILE" --local-dir "$MODEL_DIR/checkpoints" \
               --local-dir-use-symlinks False 2>/dev/null; then
            mv "$MODEL_DIR/checkpoints/$MODEL_FILE" "$CKPT"
            log "   模型下载完成（hf-mirror）"
        else
            # ③ 正源
            log "   镜像均失败，回退 HuggingFace 正源 ..."
            huggingface-cli download runwayml/stable-diffusion-v1-5 \
                   "$MODEL_FILE" --local-dir "$MODEL_DIR/checkpoints" \
                   --local-dir-use-symlinks False
            mv "$MODEL_DIR/checkpoints/$MODEL_FILE" "$CKPT"
        fi
    fi
else
    log "9. 模型已存在，跳过"
fi

# 11. 软链接模型目录
ln -sfn "$MODEL_DIR" "$COMFY_DIR/models"

# 12. 生成启动脚本
cat > "$WORK_DIR/start.sh" <<'EOF'
#!/usr/bin/env bash
source ~/miniforge3/bin/activate
conda activate comfy
cd "$(dirname "$0")/ComfyUI"
python main.py --listen --port 8188
EOF
chmod +x "$WORK_DIR/start.sh"

log "✅ 全部完成！"
log "工作目录：$WORK_DIR"
log "启动方式：bash $WORK_DIR/start.sh"
log "浏览器访问：http://127.0.0.1:8188"
```


### 为什么用python而不是编译好的二进制程序？

comfyUI是纯python项目，核心逻辑、节点系统、模型加载器全部用 PyTorch + Python 写成。没有 Go/Rust/C++ 编译型主程序，入口就是  main.py

### 为什么需要用huggingface_hub而不是wget来下载？

官方/社区只把模型托管在 Hugging Face Hub（HfHub）上，而 HfHub 的直链每次 302 跳转到云存储，且 15 分钟后失效。wget 只能拿到一个过期的 302 地址，于是出现「今天脚本还能下，明天就 404」——这正是你前面屡次遇到的情况。

huggingface_hub  CLI/库的优势：
- 自动处理 临时 Bearer token + 302 跳转，永不过期。
- 支持 断点续传、校验 SHA256，大文件下到 99% 断网也不怕。通常都是大文件，更体现其必要性
- HF 镜像站（hf-mirror）同样兼容这套 API，所以脚本里  export HF_ENDPOINT=https://hf-mirror.com  就能走国内加速。所以脚本会出现  pip install huggingface_hub  +  huggingface-cli download
- 一行命令即可指定文件名、版本号，未来换模型不用改脚本。

ModelScope 提供长期 CDN 直链，不会 302 失效（更推荐用 modelscope download）
```bash
# 官方 SD1.5 精简版（固定直链）
wget -c https://www.modelscope.cn/models/AI-ModelScope/stable-diffusion-v1-5/resolve/master/v1-5-pruned-emaonly.safetensors -O sd15_2g.safetensors
```

ModelScope (download)
作用：阿里开源镜像站提供的「国内 CDN 直链」下载工具；文件长期固定，不走 302。
HF 镜像一旦 404，立刻切 ModelScope 固定直链，仍可  wget -c  断点续传。
命令行  modelscope download  等价于「永久直链 + 断点续传 + 自动校验」。


### 其他延伸阅读

ComfyUI/models
├── checkpoints          ← 主模型（SD 1.5、SDXL、二次元、写实...）
├── loras                ← 小补丁模型（LoRA、LyCORIS）
├── vae                  ← 颜色/细节「调色盘」
├── controlnet           ← 控制网络（边缘、深度、姿态、线条）
└── clip_vision          ← CLIP 视觉编码器（IP-Adapter、风格迁移用）


| 概念                            | 含义                                                              | 文件特征                                                     | 作用                                                                                                                                                    |
| ----------------------------- | --------------------------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| checkpoints                   | 完整 SD 模型，俗称「底模」「大 ckpt」                                         | <br>体积最大（1.x – 7 GB），后缀  .ckpt  或  .safetensors          | 命名规律：<br>- v1-5-pruned  = 官方 SD1.5 全权重<br>-  v1-5-pruned-emaonly  = 去掉非 EMA 权重，体积减半，画质几乎一样<br>-  .safetensors  vs  .ckpt  = 相同权重，不同格式；前者无反序列化风险，加载更快。 |
| loras（LoRA / LyCORIS）         | Low-Rank Adaptation，「低秩补丁」。只改底模里 1–200 MB 的矩阵，就能让角色/画风/物体呈现新特征。 | 几十 MB 级，文件名常带  lora 、 locon 、 lokr                       | 用法：在 ComfyUI 里用 LoRA Loader 节点挂到主模型后面，权重 0–1 可调。                                                                                                      |
| vae（Variational Auto-Encoder） | 负责「潜空间 ↔ 像素空间」编解码的组件，影响颜色饱和度和微细节                                | 几十到几百 MB；官方名 vae-ft-mse-840000-ema-pruned.safetensors    | 底模自带 VAE，但你可以换「-ft-mse」版让颜色更鲜艳；或在 ComfyUI 里用 VAE Loader 节点单独加载                                                                                        |
| controlnet                    | 控制网络，把边缘/深度/姿态/线条图注入生成过程，实现「结构保真」                               | 几百 MB–1.4 GB；名字带  control_v11p_sd15_xxx                  | 用法：ComfyUI Load ControlNet 节点 + 对应预处理器（边缘提取等）                                                                                                         |
| clip_vision                   | CLIP 视觉编码器，用于 IP-Adapter、风格参考图、face-ID 等「图生图」高级玩法               | 几十到几百 MB；常见  CLIP-ViT-H-14-laion2B-s32B-b79K.safetensors | 用法：IP-Adapter 节点里选它，把参考图编码成向量注入生成流                                                                                                                    |

### 模型文件名举一反三说明

举例：https://www.modelscope.cn/models/AI-ModelScope/stable-diffusion-v1-5/files
这里会看到几个模型文件：

| 文件名                             | 尺寸    |
| ------------------------------- | ----- |
| v1-5-pruned-emaonly.ckpt        | 4.27G |
| v1-5-pruned-emaonly.safetensors | 4.27G |
| v1-5-pruned.ckpt                | 7.7G  |
| v1-5-pruned.safetensors         | 7.7G  |
说明：	
- v1-5-pruned.ckpt / .safetensors	完整权重，7.7 GB，最大，除非特殊需求	
- v1-5-pruned-emaonly.ckpt / .safetensors	去掉非 EMA，4.27 GB，推荐，画质几乎一样，体积减半
- .safetensors vs .ckpt 格式差异，权重相同，优先 `.safetensors`（加载快、无安全隐患）	

所以：下  v1-5-pruned-emaonly.safetensors  即可，放进  checkpoints ，ComfyUI 里直接选它当底模


### 后续

学会在 https://hf-mirror.com/ 这些网站上找到模型，有些大有些小，下载完放在 /work/llm-devlab/models/checkpoints/

学会用 huggingface、modelscope download、conda等去下载需要的组件


### 关于 mps和cuda

先确认 PyTorch 已经带 MPS
```bash
# 在 comfyui 的 venv 里执行
python - <<'PY'
import torch, platform
print(torch.__version__, platform.mac_ver()[0])
print("mps available:", torch.backends.mps.is_available())
print("mps built:", torch.backends.mps.is_built())
PY
```
只要两行都是 True 就继续；如果 False，先升级/重装 pytorch：
```bash
# 先 deactivate 再重新激活 venv，确保干净
pip uninstall -y torch torchvision
# 2024-10 之后的 nightly 才完整支持 M4，建议直接装 nightly
pip install --pre torch torchvision torchaudio --index-url \
  https://download.pytorch.org/whl/nightly/cpu
```

MPS 对应 Apple M 系列，CUDA 对应 NVIDIA
- CUDA 只跑在 NVIDIA 的闭源驱动 + cuDNN/cuBLAS 上。
- MPS 是 Apple 的 Metal API 子集，仅 macOS/iOS 存在。
PyTorch 分别做了两套后端，装对应版本即可
- 官方 nightly 已经把 MPS 和 CUDA 编进同一条  torch.whl ，只是运行时根据硬件自动暴露  torch.cuda  或  torch.mps 。
 - 在 Linux+NV 的机器上  torch.cuda.is_available()  为 True；在 macOS+M 系列上  torch.mps.is_available()  为 True，两者不会同时 True。
 - 基座（底模）是依赖于pytorch的，不需要和mps/cuda做耦合
 - comfyui虽是上层web应用，主程序已自动适配pytorch，但一些第三方节点可能会硬编码cuda()等，导致RuntimeError

以下认知都正确吗？
mps是对应苹果的m系列芯片，cuda是对应nvidia的芯片；
pytorch分别做了应对MPS和CUDA的不同分支处理，依赖时用相应的版本即可；
基座模型（底模）底层是用pytorch的，所以大模型不需要处理MPS和CUDA的问题；
ComfyUI作为上层的web应用，所以也不需要处理mps和cuda