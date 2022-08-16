# Neo-Express介绍
Neo-Express是一种方便开发为目的设置本地私有区块链的工具，可以在Visual Studio Code上通过添加扩展的方式来进行添加。使用私有节点更方便我们进行调试，以及取之不尽的GAS，等代码稳定后，再放入测试网进行测试，再将合约发布到主网。

# 相关下载
- [**.NET SDK**](https://dotnet.microsoft.com/download)
- [**Visual Studio Code**](https://code.visualstudio.com/)
- [**Neo Blockchain Toolkit**](https://marketplace.visualstudio.com/items?itemName=ngd-seattle.neo-blockchain-toolkit)
- [**Neo N3 Visual DevTracker**](https://marketplace.visualstudio.com/items?itemName=ngd-seattle.neo3-visual-tracker)
- [**相关视频参考**](https://developers.neo.org/tutorials/2021/05/27/getting-started-with-the-neo-blockchain-toolkit)

# 注意
Neo Blockchain Toolkit相当于一个下载器，所以它的版本号是不会更新的，下载器会去下载Visual DevTracker的最新版本，当前项目中使用的是V3.1.25版本，后续的版本支持配置session.解决了最多返回100 items的issue.
当合约执行结果里包含迭代器 Iterator 类型时，会根据 RpcServer 插件的 config.json 文件中 SessionEnabled 字段的值来决定是否返回 session ，如果 SessionEnabled 为 true ，返回结果中会包含 session 字段，用来在 traverseiterator 方法中进一步获取 Iterator 的详细内容，如果 SessionEnabled 为 false ，则返回结果中不包含 session ，无法进一步获取 Iterator 的详细内容,请注意相应的版本号，可能导致当前项目可能无法读取数据，需要修改相应代码。