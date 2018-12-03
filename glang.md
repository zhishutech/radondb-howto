# golang安装

因为RadonDB是使用Go语言开发的，所以在运行的时候需要依赖Go环境

* 下载Go-1.11.2安装包

```
# wget https://dl.google.com/go/go1.11.2.linux-amd64.tar.gz
```

* 解压安装包

```
# tar -xvf go1.11.2.linux-amd64.tar.gz -C /usr/local
```

* 修改环境变量

```
在~/.bash_profile末尾添加一行
export PATH=$PATH:/usr/local/go/bin

# source ~/.bash_profile
```

* 验证Go安装是否成功

```
# go version
go version go1.11.2 linux/amd64
```