

###### 关于魔搭社区notebook使用

你遇到的是魔搭目前“强制路由到阿里云 PAI-DSW”的新规则，与手机号是否绑定过阿里云无关。
只要站点检测到「中国大陆 IP + 未实名」就会默认走 PAI-DSW 通道，并弹出“关联阿里云账号”提示；不再像去年那样给纯粹的「ModelScope Notebook」入口。
换句话说——
现阶段（2024-12 以后）所有新注册账号都只能在 PAI-DSW 里开实例；
真正的「ModelScope Notebook（每天 8h 免费 P40）」已暂停对外放号，老账号之前开过机的还能续用，新人看不到。
现状与可行路线
1. 
继续留在 PAI-DSW（唯一官方通道）
 
免费部分
– CPU：8 核 32 G，长期可用，但跑 ComfyUI 基本卡死。
– GPU：初始送「3h A10 额度」+ 每日签到再领 1h，可攒到 6h。
 
操作步骤
① 勾选“全部授权”→ 关联阿里云（仅实名，不收费）→ 立即领券。
② 创建实例 → 镜像选“官方镜像-ComfyUI”→ 开机 → 浏览器侧边栏点「打开」即可用。
 
出图速度：A10 上 SD1.5/20 步 512×512 约 8s，FLUX-dev-fp16 约 45s。
 
券用完就关机，第二天再领；想多玩可 9.9 元买 10h 券（学生价 4.9 元）。
2. 
想回“纯魔搭 8h”——目前官方已关闭新入口，无法恢复；只能：
 
用老账号（2023-10 前注册且开过 ModelScope Notebook）继续续命；
 
等官方再次放号（社区公告未给出时间表）。
3. 
完全不想关联阿里云 → 换境外源
 
Colab / HuggingFace Spaces / 端脑云，都提供 ComfyUI 模板；
 
国内网络需科学环境，否则模型拉取慢。
结论（可执行）
 
新人现在只有“PAI-DSW”这一条路，点“关联阿里云”只是实名，不扣费；领完券就能开 GPU 玩 ComfyUI。
 
如果你确实“零成本 + 零绑定”，那就用 CPU 实例装轻量 SD1.5-gguf（一张图 5-8 min 当体验），或者干脆转战 Colab。
 
日出图 <30 张的娱乐需求，把免费 3h 券攒成 6h 后，每周集中玩一次即可，依旧 0 元。
一句话：“ModelScope 纯 8h 入口”已对新用户下线；当下想白嫖 GPU 跑 ComfyUI，就乖乖走 PAI-DSW 领券，别的捷径暂时没有。

###### 体验流程

1. 先选 PAI-DSW -> CPU环境
	 - 能做什么
		 - 验证工作流 json 是否能正常加载、节点是否缺依赖。
		 - 生成 256×256 小图做“跑通测试”。
		 - 写脚本批量跑「文本-embedding」或「预处理」步骤，这些纯 CPU 任务反而合适。
	- 不能做什么
		 - 正常尺寸实时出图、调参、LoRA 训练。
		 - FLUX / SDXL 高清图；一张图半小时，娱乐感全无。
	- 结论
		- “只是让它跑起来看看界面” → CPU 可以。
		- “想体验 ComfyUI 出图快感” → 必须上 GPU（方式一），哪怕 3h 券用完关机，也比 CPU 熬半小时强。 

---

touch sys-check.sh
vi sys-check.sh

# 保存为 sys-check.sh 后直接 bash sys-check.sh
## TODO 需要优化


#!/bin/bash
out=/mnt/workspace/system_snapshot.md
# 如文件不存在先创建空表头
[ ! -f "$out" ] && echo "# 系统快照 $(date '+%F %T')" > "$out"

{
cat <<'EOF'
### 1. 容器信息，确认是 K8s Pod 还是 Docker
| 命令 | 结果 | 说明 |
|----|----|----|
EOF
printf "| \`cat /proc/1/cgroup \\| head -1\` | \`\`\`%s\`\`\` | 含 kubepods / docker / cri-containerd |\n" "$(cat /proc/1/cgroup | head -1)"
printf "| \`ls /.dockerenv\` | \`\`\`%s\`\`\` | 文件存在 → Docker |\n" "$(ls /.dockerenv 2>/dev/null && echo "存在" || echo "不存在")"
printf "| \`ps -q 1 -o comm=\` | \`\`\`%s\`\`\` | PID1 进程名（containerd-shim / tini / bash） |\n" "$(ps -q 1 -o comm=)"
} > "$out"

######### 2. 硬件信息 #########
{
cat <<'EOF'

### 2. 硬件信息
| 命令 | 结果 | 说明 |
|----|----|----|
EOF
# 注意：printf 里把命令两侧的 ` 保留，内部 | 不需要转义；结果用 %s 回填
printf "| \`lscpu \| grep \"Model name\"\` | \`\`\`%s\`\`\` | CPU 型号 |\n" "$(lscpu | grep "Model name" | cut -d: -f2- | xargs)"
printf "| \`nproc\` | \`\`\`%s\`\`\` | 物理核数 |\n" "$(nproc)"
printf "| \`free -h \| awk '/^Mem:/ {print \$2}'\` | \`\`\`%s\`\`\` | 总内存 |\n" "$(free -h | awk '/^Mem:/ {print $2}')"
printf "| \`free -h \| awk '/^Mem:/ {print \$3}'\` | \`\`\`%s\`\`\` | 已用内存 |\n" "$(free -h | awk '/^Mem:/ {print $3}')"
printf "| \`df -h /\` | \`\`\`%s\`\`\` | 根盘总量 |\n" "$(df -h / | awk 'NR==2 {print $2}')"
printf "| \`df -h /\` | \`\`\`%s\`\`\` | 根盘已用 |\n" "$(df -h / | awk 'NR==2 {print $3}')"
printf "| \`nvidia-smi --query-gpu=name --format=csv,noheader\` | \`\`\`%s\`\`\` | GPU 型号 |\n" "$(nvidia-smi --query-gpu=name --format=csv,noheader 2>/dev/null || echo "none")"
printf "| \`nvidia-smi --query-gpu=memory.total --format=csv,noheader\` | \`\`\`%s\`\`\` | GPU 显存总量 |\n" "$(nvidia-smi --query-gpu=memory.total --format=csv,noheader 2>/dev/null || echo "N/A")"
} >> "$out"

######### 3. 网络信息 #########
{
cat <<'EOF'

### 3. 网络信息
| 命令 | 结果 | 说明 |
|----|----|----|
EOF
printf "| \`hostname -I\` | \`\`\`%s\`\`\` | 内网 IP |\n" "$(hostname -I | awk '{print $1}')"
printf "| \`curl -s ifconfig.me\` | \`\`\`%s\`\`\` | 出口公网 IP |\n" "$(curl -s ifconfig.me 2>/dev/null || echo "offline")"
printf "| \`ip route get 1\` | \`\`\`%s\`\`\` | 默认源 IP |\n" "$(ip route get 1 | awk '{print $7}')"
printf "| \`ss -tunlp \| grep :8188\` | \`\`\`%s\`\`\` | ComfyUI 监听（如有） |\n" "$(ss -tunlp | grep :8188 || echo "N/A")"
} >> "$out"

######### 4. 进程与资源实时 #########
{
cat <<'EOF'

### 4. 进程与资源
| 命令 | 结果 | 说明 |
|----|----|----|
EOF
# 把整段输出放进 ``` ``` 避免换行破坏表格
printf "| \`ps aux --sort=-%%mem \| head -5\` | \`\`\`%s\`\`\` | 内存占用前 5 进程 |\n" "$(ps aux --sort=-%mem | head -5)"
printf "| \`ps aux --sort=-%%cpu \| head -5\` | \`\`\`%s\`\`\` | CPU 占用前 5 进程 |\n" "$(ps aux --sort=-%cpu | head -5)"
printf "| \`cat /proc/loadavg\` | \`\`\`%s\`\`\` | 平均负载（1min 5min 15min） |\n" "$(cat /proc/loadavg)"
} >> "$out"

######### 5. 调试与日志 #########
{
cat <<'EOF'

### 5. 调试与日志
| 命令 | 结果 | 说明 |
|----|----|----|
EOF
printf "| \`dmesg \| tail -10\` | \`\`\`%s\`\`\` | 内核最近 10 条 |\n" "$(dmesg | tail -10)"
printf "| \`journalctl -u jupyter -n 10 2>/dev/null\` | \`\`\`%s\`\`\` | Jupyter 日志末 10 行 |\n" "$(journalctl -u jupyter -n 10 2>/dev/null || echo "no systemd")"
printf "| \`ls -lh /var/log/ \| head -5\` | \`\`\`%s\`\`\` | 日志目录大小 |\n" "$(ls -lh /var/log/ 2>/dev/null | head -5)"
} >> "$out"

echo "✅ 完成"
