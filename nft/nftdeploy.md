# 发布到测试网，主网
>合约经过开发，测试，之后没有异常，合约一直是在NEOEXPRESS种。我们现在要将合约发布到测试网进行测试，在发布到主网。
>测试网和主网发布的过程一样，这里只演示测试网的发布过程
## 申请测试网GAS和NEO
[申请测试网GAS和NEO地址](https://n3t5wish.ngd.network/)
## 发布合约到测试网
打开合约程序[合约部署](nft/contract.md),` com.axlabs.boilerplate.Deployment`中将Node地址切换到测试网
```java
private static final String NODE = "http://seed1t5.neo.org:20332";
// private static final String NODE = "http://localhost:50012";
```
修改完毕后`run`,由于合约没有进行更改，所以生成的合约地址和NEOEXPRESS是一样的。
部署完毕后查看详情：

[测试网查看部署详情](https://testmagnet.explorer.onegate.space/contractinfo/0x88076fb39dab509c64423e67a71a0a59e1f758ff)

修改代码中相应的NEOEXPRSS私有网络地址，为测试网。进行测试。再按照相同方法发布至主网。
```java
public class Constants {
    // This sets up the connection to the neo-node of our private network.
    //public static Neow3j NEOW3J = Neow3j.build(new HttpService("http://localhost:50012", true));
    //TestNet
    public static Neow3j NEOW3J = Neow3j.build(new HttpService("http://seed1t5.neo.org:20332", true));
```    

