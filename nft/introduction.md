# 项目介绍
- 开发一个邮票NFT的简单网站，可以在系统中，发布制作NFT邮票，可以赠送给其他用户，并且可以查询用户所拥有的NFT。

- 目的：熟练运用java来创建合约，以及通过java后台和NeoLine与合约进行交互。

- 假设一个邮票由4个属性组成， 编号(唯一)，名称， 图片，描述。我们要做的就是将这些属性存到NFT合约中，和账户建立对应关系，就可以实现不同用户拥有不同NFT的效果。

- 图片可以由3种方式存储到合约中，图片存储形式：
 1. 存储到中心化的服务器中，将图片的url访问地址存储到合约中。
 2. 保存到去中心化的NeoFs中，将图片的url访问地址存储到合约中。
 3. 将图片格式化成Base64直接存储到合约中（需要足够的小才行）。
 ## 完整效果
<video id="video" controls="controls" preload="none"  width="800" height="600">
      <source id="mp4" src="./images/nftsystem/20220804101028.mp4" type="video/mp4">
</video>

点击播放按钮播放，全屏按钮全屏观看

