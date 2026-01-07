- 是什么？
	- 一个用go写的程序，所以是可以直接运行的二进制

### 安装/运行 (mac系统)

 - 安装
	- brew install ollama
- 启动（后台常驻）
	- ollama serve
- 拉取模型
	- ollama pull qwen2:0.5b
	- （中文好，轻量，仅300M，但后面的权重GGUF还是要4.4GB）
- webUI
	- open http://localhost:11434
	- （仅显示 ollama is running，确认ollama正常运行，并非已完成了大模型的部署）


ollama pull  的默认行为是去 Ollama 官方仓库（ registry.ollama.ai ）拉取，比如 ollama pull tinyllama 实际地址为 registry.ollama.ai/library/tinyllama 官方已打包好的镜像，若需要国内加速，ollama pull 时直接写国内镜像地址即可，无需额外设置；不写就默认走官方 registry.ollama.ai

###### 举例：

| 镜像源         | 命令示例                                                                                          |
| ----------- | --------------------------------------------------------------------------------------------- |
| ModelScope  | `ollama pull modelscope.cn/Qwen/Qwen2.5-0.5B-Instruct-GGUF:qwen2.5-0.5b-instruct-q4_k_m.gguf` |
| HF 镜像站      | `ollama pull hf-mirror.com/unsloth/TinyLlama-1.1B-Chat-GGUF:Q4_K_M`                           |
| DeepSeek 镜像 | `ollama pull ollama.deepseek.com/deepseek-r1:1.5b`                                            |

###### 运行大模型

ollama run qwen2
	然后会进入漫长的下载，因为有4.4G
	 换成 ollama run qwen2:0.5b 则会快很多，甚至可以马上进入对话，对hello world级体验很好

✅ 可以多终端同时  ollama run 模型 ，即使同名模型也支持并发对话； 
⚠️ 不要手工起多个  ollama serve ，官方设计是“单服务 + 多模型/多并发”，再启第二个实例会抢端口或锁库，容易出错。

并发上限由环境变量控制，默认即可满足 3-4 路同时对话；想再提高：
```shell
export OLLAMA_NUM_PARALLEL=8          # 单模型并发路数
export OLLAMA_MAX_LOADED_MODELS=4     # 同时驻留内存的模型份数
ollama serve
```
