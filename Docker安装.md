
### 我的机器环境
宿主：MacBook-air-m4 16+256 10+8
系统：macos sequoia 15.6

###### 下载：
	https://www.docker.com/get-started/ 
	download docker desktop -> download for Mac - apple silicon

安装后，提示需要安装rosetta，安装即可

查看版本：
```bash
docker version
```
Client:
 Version:           28.5.2
 API version:       1.51
 Go version:        go1.25.3
 Git commit:        ecc6942
 Built:             Wed Nov  5 14:42:30 2025
 OS/Arch:           darwin/arm64
 Context:           desktop-linux

Server: Docker Desktop 4.51.0 (210443)
 Engine:
  Version:          28.5.2
  API version:      1.51 (minimum version 1.24)
  Go version:       go1.25.3
  Git commit:       89c5e8f
  Built:            Wed Nov  5 14:44:06 2025
  OS/Arch:          linux/arm64
  Experimental:     false
 containerd:
  Version:          v1.7.29
  GitCommit:        442cb34bda9a6a0fed82a2ca7cade05c5c749582
 runc:
  Version:          1.3.3
  GitCommit:        v1.3.3-0-gd842d771
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0

###### 配置国内镜像源：
Settings → Docker Engine
将右侧 JSON 内容全部替换为下方配置（已按优先级排序，多源容错）
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://docker-0.unsee.tech",
    "https://docker.xuanyuan.me",
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
  ],
  "experimental": false,
  "features": {
    "buildkit": true
  }
}
点击 Apply & Restart，Docker 会自动重启，30 秒后生效