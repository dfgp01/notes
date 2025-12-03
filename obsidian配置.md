
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

github配置
以我的mac notes项目为参考：

```bash
# 进入目标目录
cd /Users/hugo/work/notes

# 初始化仓库
git init

# 添加远程地址（HTTPS 示例；如习惯 SSH 把地址换成 SSH 即可）
git remote add origin https://github.com/xxx/notes.git

# 新建 .gitignore（可选，排除 Ob 缓存）
cat <<EOF > .gitignore
.obsidian/*
EOF

# 首次提交
git add .
git commit -m "first commit"

# 推送到主分支
git branch -M main
git push -u origin main
```

github配置personal Access Token（PAT）

若推送时，用户名密码都无误的情况下出现 “Password authentication is not supported” ，是因为GitHub 早已禁止“账号+密码”直接推代码，必须用 Personal Access Token（PAT） 或 SSH Key。


1. 生成 PAT
浏览器 → 登录 GitHub → 右上角头像 → Settings → 左侧 Developer settings → Personal access tokens → Tokens (classic) → Generate new token
 
Token name: obsidian-used
Description:  only obsidian notes
Expiration: 90 days
Repository access: Only select repositories -> xxx/notes
Permissions: actions -> read and write; Administration -> read and write
 （目前是全仓全权且读写都开，因为独仓会403，原因不明）
点击 Generate token  出现 github_pat_*********  字符串（只显示一次）

2. 把旧 remote 清掉重建（避免缓存）
```bash
cd /Users/hugo/work/notes
git remote remove origin
git remote add origin https://github.com/xxx/notes.git
```

3. 重新推送，密码处粘贴 Token
```bash
git push -u origin main
```
弹出钥匙串窗口：
Username 填 GitHub 账号名（不是邮箱）
 Password 填刚才复制的 Token（含 ghp*）
成功后 Git 会把 Token 存进 macOS 钥匙串，以后一键推送无需再输
而且github推送时不允许提交隐私信息，因此我把相关都用XXX替代

sourcetree中的设置：
设置 -> 高级 -> 编辑配置文件
模板如下（只需改  TOKEN  和  USER ）：
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
	ignorecase = true
	precomposeunicode = true

[remote "origin"]
	url = https://TOKEN@github.com/USER/notes.git
	fetch = +refs/heads/*:refs/remotes/origin/*

[credential]
	helper = store
