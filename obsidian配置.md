
安装插件（需解除安全模式）

文件同步
前提：本地已安装git
步骤：设置 -> 社区插件 -> 浏览 -> 搜索git -> 安装 -> 启用


AI助手
前提：需要有相应大模型的API，这里举例kimi和deepseek
步骤：设置 -> 社区插件 -> 浏览 -> 搜索copilot -> 安装 -> 启用
	设置 → Copilot → Model -> ChatModels -> Add Model
	 Name：kimi-k2-turbo-preview
	 Provider：OpenAI
	 Base URL：https://api.moonshot.cn/v1（兼容OPENAI）
	 API-KEY：在各自大模型供应商中获取
	 备注：deepseek在copilot中内置，只需修改参数，而自定义model里的provider是预设的选项，没有moonshot，所以这里用openai，目前没运行成功

github配置：
	已放弃obsidian内的git方式，因为github的http操作比较麻烦，我更偏向于独立的git使用，如用sourcetree等软件进行，毕竟同步操作并不频繁。
