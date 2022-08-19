# 发布到测试网，主网
>合约经过开发，测试，之后没有异常，合约一直是在NEOEXPRESS种。我们现在要将合约发布到测试网进行测试，在发布到主网。
>测试网和主网发布的过程一样，这里只演示测试网的发布过程
## 申请测试网GAS和NEO
[申请测试网GAS和NEO地址](https://neowish.ngd.network/)
## 发布合约到测试网
打开合约程序[合约部署](nft/contract.md),` com.axlabs.boilerplate.Deployment`中将Node地址切换到测试网
```java
private static final String NODE = "http://seed1t4.neo.org:20332";
// private static final String NODE = "http://localhost:50012";
```
修改完毕后`run`,由于合约没有进行更改，所以生成的合约地址和NEOEXPRESS是一样的。


