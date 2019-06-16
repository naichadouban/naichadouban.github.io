p2p协议

# 1.介绍

> p2p网络协议是一个不断变化的过程，并没有固定的规格。peer之间的通信完全是基于tcp

## 一些常量

| [Network](https://bitcoin.org/en/p2p-network-guide)          | Default Port | [Start String](https://bitcoin.org/en/glossary/start-string) | Max [nBits](https://bitcoin.org/en/glossary/nbits) |
| ------------------------------------------------------------ | ------------ | ------------------------------------------------------------ | -------------------------------------------------- |
| [Mainnet](https://bitcoin.org/en/glossary/mainnet)           | 8333         | 0xf9beb4d9                                                   | 0x1d00ffff                                         |
| [Testnet](https://bitcoin.org/en/glossary/testnet)           | 18333        | 0x0b110907                                                   | 0x1d00ffff                                         |
| [Regtest](https://bitcoin.org/en/glossary/regression-test-mode) | 18444        | 0xfabfb5da                                                   | 0x207fffff                                         |

`Port` 在节点启动的时候可以自己配置

`Start Sring` 是硬编码到代码中的，所有通过比特币网络传输的消息，起始字符串都是上面定义的`Start String`,还有存储区块数据库的起始字符串可能也是它。

`nBits`：上面显示的是大端，通过网络传输时是小端。

其他常量，暂时不讨论。

## 协议版本

peer之前通信，握手前先要发送版本信息，先要看两者是不是同一个版本的。

