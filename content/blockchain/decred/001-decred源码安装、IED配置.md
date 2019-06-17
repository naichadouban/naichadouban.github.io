---
title: decred源码环境/IDE配置
date: 2019-06-13T00:00:00+08:00
tags: ["decred"]
categories: ["decred"]
draft: false

---

本教程不面向初学者, 面向区块链程序员.

默认你已经熟悉了BTC, Decred, 比如你已经掌握了以下知识:

*   掌握 Golang 语言
*   看过 *[精通比特币](http://book.8btc.com/books/6/masterbitcoin2cn/_book/),* 而且最好看了至少2遍
*   熟悉 UTXO 模型
*   熟悉 Decred PoS 机制, 选票几种状态熟悉
*   拥有 BTC/Decred, 买过票

# 1. Decred源码环境/IDE配置
## 安装Golang
最新源码支持go module, 需要Golang 1.11或以上版本. 安装的教程很多, 这里不再赘述.
## 下载Decred源码并安装依赖
请至 https://github.com/decred/dcrd 下载最新主分支代码, 推荐使用git clone的方式, 因为以后在看源码的时候难免会修改一些代码, 这样通过git可以很方便看到修改的地方, 另外通过git方式也方便以后同步最新代码.
```
cd ~/ # 切换到任何一个你想存储dcrd代码的目标, 不必是$GOPATH/scr/下
git clone https://github.com/decred/dcrd.git dcrd-study # dcrd-study这个名字请随意取

```
打包, go install会先自动下载依赖
```
cd dcrd-study
go install . ./cmd/...

```
这一步可能由于网络原因不能成功, 如你有可能出现以下信息:
```
go install . ./cmd/...
go: finding github.com/decred/dcrwallet/rpc/jsonrpc/types v1.0.0
go: golang.org/x/sys@v0.0.0-20190203050204-7ae0202eb74c: unrecognized import path "golang.org/x/sys" (https fetch: Get https://golang.org/x/sys?go-get=1: dial tcp 216.239.37.1:443: i/o timeout)
go: error loading module requirements

```
因为dcrd依赖 `golang.org/x/crypto` , 而在国内这个库因为莫名其妙或众所皆知的原因是不能访问的. 有以下几种方法解决:
方法一, 使用科学上网. 这里不展开.
方法二, 使用GOPROXY.
Go module 支持通过代理的方式下载, 如果环境变量 GOPROXY 设置了, 所有的包都会从这个代理下载.
代理基于 HTTP 协议的 GET 方法, 请求的时候没有参数, 所以只要是符合固定的规则, 任何服务器都可以做代理服务器. 
Go module 的代理比较出名的有微软的 [athens](https://github.com/gomods/athens), 可以基于它搭建一个私有的代理, 管理内部的私有代码, 而且微软提供了一个公共的代理 https://athens.azurefd.net 我们可以直接使用.
下面设置代码, 并build dcrd.
```
export GOPROXY="https://athens.azurefd.net"
go install . ./cmd/...

```
如果没报错, 证明打包成功, 打包的二进制文件在配置的$GOPATH/bin下.
这里大家可能会对 `go install . ./cmd/...`有疑问, 这里面有几个点是什么意思? 其实这个命令包含两个部分,  `.`和`./cmd...`是两个不同的目录, `.`是用来编译当前目录的, 也就是编译dcrd命令, 可以单独执行 `go install .`. `./cmd/...`表示cmd下的所有目录, 在`cmd/`下有`dcrctl/, addblock/, findcheckpoint/, gencerts/, promptsecret/`, 通过 `go install ./cmd/...`这一个命令就编译了`cmd/`所有子文件夹. 当然, 你也可以单独编译某个目录, 比如我们单独编译dcrctl, `go install ./cmd/dcrctl`.
还有, 如果你不希望将编译的二进制文件放在 $GOPATH/bin下, 而是在当前目录下, 你可以使用 `go build`, 如 `go build .`, 这样就在当前目录生成了dcrd二进制文件. 同理, 你也可以编译cmd里的命令.
## 安装Goland
看源码或开发离不开一个好的IDE, 在Golang前些年是没有一个好用的IDE的, 如之前我用过的 goclipse, sublime, LiteIDE, 这些都不好用. 开发过 IntelliJ IDEA 的 [JetBrains](http://www.baidu.com/link?url=fwdONV05WTQI3eySs5VF5r92AtZba9FYlhwughZfp8-tZDuzx2B7wx5G9xyGdRez) 公司开发了Golang的IDE Goland 秒杀其它所有IDE. 这里也推荐大家使用 Goland, 真心好用, 品质之作.
请至 https://www.jetbrains.com/go/ 下载并安装. 当然它不是一个免费的产品, 要$199.00/年, 有条件的朋友一定要支持正版! 没条件的朋友可以先试用或者寻求破解方法.
## 使用Goland
打开后, 新建项目:
左侧选择 "Go Modules (vgo)",  右侧Location选择刚才git clone下的dcrd目录. GOROOT一般会默认你安装Golang时设置的GOROOT. 然后点击Create.
![image.png](https://upload-images.jianshu.io/upload_images/422094-27e071f428195cfc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


Create会提示是否以所选目录下的代码来创建项目, 点击 "Yes"
![image.png](https://upload-images.jianshu.io/upload_images/422094-38f4839da5ae8a1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

进入主界面后, 可能是Goland的bug, 并没有启动Go Module, 我们点击Goland-&gt;Preferences. 按下图设置, 并Apply.
![image.png](https://upload-images.jianshu.io/upload_images/422094-a4d8e5f3ab8db0ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Goland会对dcrd源码建索引, 这会耗几分钟时间.
小试Goland:
打开 blockchain/fullblocks_test.go, 找到 TestFullBlocks 点击左侧的运行按钮运行该测试方法. 之后在学习源码的过程中会经常用到运行测试用例的情况, 而且我们也可以通过查看测试用例学习代码间的关联和如何使用. 我们以后也会写一些测试用例来调用其它方法进行学习.
![image.png](https://upload-images.jianshu.io/upload_images/422094-8439232286f93360.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





