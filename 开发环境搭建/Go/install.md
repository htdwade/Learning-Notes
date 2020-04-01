## 1. Go下载安装

[Go下载地址](https://golang.google.cn/dl/)

### Windows

```bash
//下载MSI文件安装
//创建GoPath目录，里面创建bin，pkg，src三个文件夹
//设置环境变量
GOROOT E:\Go
GOPATH E:\GoPath
Path E:\Go\bin; E:\GoPath\bin

// 设置代理
go env -w GOPROXY=https://goproxy.cn,direct

// 检查是否安装成功
go version
```

### Linux

```bash
# 下载解压
wget https://dl.google.com/go/go1.13.5.linux-amd64.tar.gz
tar zxvf go*.tar.gz -C /usr/local/

# 配置环境：注意该环境必须是go1.11版本及以上且项目要求使用go mod才可以开启
vim /etc/profile
export GOROOT=/usr/local/go                 # golang本身的安装位置
export GOPATH=~/go                          # golang包的本地安装位置
export GOPROXY=https://goproxy.io,direct    # golang包的下载代理,回源地址获取
export GO111MODULE=on                       # 开启go mod模式
export PATH=$PATH:$GOROOT/bin               # go本身二进制文件的环境变量
export PATH=$PATH:$GOPATH/bin               # go第三方二进制文件的环境便令

# 重启环境
source /etc/profile 
```
测试安装
```bash
# 查看go版本
go version

# 查看go环境配置
go env  
```

## 2. IDE

VSCode

[VSCode配置Go开发环境](https://code.visualstudio.com/docs/languages/go)

[GoLand下载地址](https://www.jetbrains.com/go/download/#section=windows)

### 汉化

[汉化教程](https://github.com/pingfangx/TranslatorX/wiki)