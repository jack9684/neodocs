# 项目介绍
本示例搭建了一个由4个共识节点组成的多节点区块链网络以及一个额外的普通节点用来进行转账操作。

其中4个节点的公钥分别为：

```
02017e169b71166aabcc1e1fb1c978d30b2fddd6099cef8745e70a8916c505a2fb
02ca31eed0c7225e57c454e66cde82a52d7b12ea768f0e181424df51c38f79cc1e
027faab0d837770234a927af25db6acd615da707d99fc2f2d30d152329f9f605ba
03f515972a056d7ee94903f9c4f8b2bfc29644aefc0e93178dfa0cd3be27e27511
``` 
4个公钥组成的多方签名地址为：
```
NWYYKni5n3QqkQsqY8WTpY84bv2Hr7NMkk
NEO: 100000000
GAS: 52000000
```
创建新的钱包地址，并从多签地址中转出10000000Gas给新地址，新地址为：
```
NKxWyrGJHKD8hFyTtng7A4JcAsnVLu31Rg
```
# 创建快捷启动 

为了方便启动私链，创建一个记事本文件，输入以下命令：
```
start cmd /k "cd c1\neo-cli  &&ping localhost -n 3 > nul&& dotnet neo-cli.dll"
start cmd /k "cd c2\neo-cli &&ping localhost -n 3 > nul&& dotnet neo-cli.dll"
start cmd /k "cd c3\neo-cli &&ping localhost -n 3 > nul&& dotnet neo-cli.dll"
start cmd /k "cd c4\neo-cli &&ping localhost -n 3 > nul&& dotnet neo-cli.dll"
```
然后重命名为 Run.cmd。将其复制到 4 个节点目录外的同级目录下运行。

# 完整效果

<video id="video" controls="controls" preload="none"  width="800" height="600">
      <source id="mp4" src="./images/privatechain/20230110135509.mp4" type="video/mp4">
</video>

点击播放按钮播放，全屏按钮全屏观