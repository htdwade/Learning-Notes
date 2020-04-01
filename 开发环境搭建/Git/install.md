## 1. Git下载安装

[Git下载地址](https://git-scm.com/downloads)

默认安装即可

![](https://raw.githubusercontent.com/htdwade/PicBed/master/img/20191214142143.png)

## 2. 环境设置

```bash
// 设置用户信息
git config --global user.name "htdwade"
git config --global user.email "1046217644@qq.com"

// 列出所有的配置信息
git config --list
// 检查某一项配置
git config <key>

```

## 3. 生成ssh密钥

```bash
// 生成ssh公钥和私钥
ssh-keygen -t rsa -b 4096 -C "1046217644@qq.com"
// 一路回车
// 将公钥复制到剪贴板
clip < ~/.ssh/id_rsa.pub
```

进入`github`设置界面，新增ssh密钥：

![](https://raw.githubusercontent.com/htdwade/PicBed/master/img/20191214152428.png)



测试ssh连接：

```bash
ssh -T git@github.com
//输入yes即可
```

