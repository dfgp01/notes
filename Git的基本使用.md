本地安装git，直接装在宿主环境而不是docker之类的虚拟化环境，因为工作所需的文件编辑和同步都在本地环境里完成（如IDE）

github, gitlab等软件平台，都是基于git上面搭建的，因此熟悉git的用法即可熟悉这些平台的操作

### 简易步骤

1. 确保本地已安装git
2. github上创建自己的仓库，公有私有随意，此过程略
3. bash命令操作
	1. ssh-keygen -t rsa -b 4096 -C "你的邮箱@example.com"，一路回车（默认保存路径是  ~/.ssh/id_rsa ），不设置密码也可以。密钥文件保存在：/Users/你的用户/.ssh，id_rsa是私钥，id_rsa.pub是公钥
	2. 打开 GitHub → 右上角头像 → Settings → SSH and GPG keys → New SSH key；Title：随便写，比如  MacBook Pro ，Key：粘贴id_rsa.pub的内容，最后 Add SSH key 即可
	3.  测试key是否成功: ssh -T git@github.com 第一次会提示是否继续连接，输入  yes 。如果看到类似：Hi 你的用户名! You've successfully authenticated...说明成功！
	4. 克隆仓库：git clone git@github.com:你的用户名/你的仓库.git（在github中有提供完整链接，直接拷贝即可），或者在你本地已有仓库的情况下，添加远程地址：git remote set-url origin git@github.com:你的用户名/你的仓库.git
	5. 然后就可以进行 git commit, git push等操作了

这里使用的是github的ssh操作方式，若使用http方式，则还需要进行一系列的操作，如PAT授权等，比较麻烦，有被这些事情耽搁了，所以一开始不需要弄这么麻烦，简单不要影响主要工作进程就好