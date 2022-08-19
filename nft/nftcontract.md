# NEP-11 (NFT)合约开发

[这是nep-11的规范](https://github.com/neo-project/proposals/blob/master/nep-11.mediawiki)一定要遵守。
## 完整代码
```java
package com.axlabs.boilerplate;

import io.neow3j.devpack.*;
import io.neow3j.devpack.Runtime;
import io.neow3j.devpack.annotations.*;
import io.neow3j.devpack.constants.CallFlags;
import io.neow3j.devpack.constants.FindOptions;
import io.neow3j.devpack.constants.NativeContract;
import io.neow3j.devpack.constants.NeoStandard;
import io.neow3j.devpack.contracts.ContractManagement;
import io.neow3j.devpack.contracts.StdLib;
import io.neow3j.devpack.events.Event2Args;
import io.neow3j.devpack.events.Event3Args;
import io.neow3j.devpack.events.Event4Args;

import java.util.HashMap;

//你合约Token的名字,如果未设置默认使用类的名字
@DisplayName("StampToken")
//额外说明信息这些信息会在manifest中看到
@ManifestExtra(key = "author", value = "jackcao")
@ManifestExtra(key = "email", value = "cjcjcjcjcj9684@163.com")
//manifest中的supportedStandards属性,代表遵循NEP11规范
@SupportedStandard(neoStandard = NeoStandard.NEP_11)
//设置你的合约允许去调用哪些合约
@Permission(nativeContract = NativeContract.ContractManagement)
public class StampToken {

    static final StorageContext ctx = Storage.getStorageContext();
    //存储已发行的NFT的tokenid
    static final StorageMap registryMap = new StorageMap(ctx, 1);
    //存储tokenid的归属关系
    static final StorageMap ownerOfMap = new StorageMap(ctx, 2);
    //存储每个用户拥有的NFT个数
    static final StorageMap balanceMap = new StorageMap(ctx, 3);

    static final byte[] contractOwnerKey = new byte[]{0x12};
    static final byte[] totalSupplyKey = new byte[]{0x10};
    static final byte[] tokensOfKey = new byte[]{0x11};

    // endregion keys of key-value pairs in NFT properties
    // region deploy, update, destroy

    @OnDeployment
    public static void deploy(Object data, boolean update) {
        if (update) return;

        Transaction Tx = (Transaction) Runtime.getScriptContainer();
        Storage.put(Storage.getStorageContext(), contractOwnerKey, Tx.sender.toByteArray());
        Storage.put(Storage.getStorageContext(), totalSupplyKey, 0);
    }

    public static void update(ByteString script, String manifest) throws Exception {
        Hash160 owner160 = Storage.getHash160(Storage.getReadOnlyContext(), contractOwnerKey);

        if (!Runtime.checkWitness(owner160)) {
            throw new Exception("No authorization.update");
        }
        new ContractManagement().update(script, manifest);
    }

    public static void destroy() throws Exception {
        Hash160 owner160 = Storage.getHash160(Storage.getReadOnlyContext(), contractOwnerKey);

        if (!Runtime.checkWitness(owner160)) {
            throw new Exception("No authorization.destroy");
        }
        new ContractManagement().destroy();
    }

    // endregion deploy, update, destroy
    // region NEP-11 methods

    @Safe
    public static String symbol() {
        return "STAMP";
    }

    @Safe
    public static int decimals() {
        return 0;
    }

    @Safe
    public static int totalSupply() {
        return Storage.getInt(Storage.getReadOnlyContext(), totalSupplyKey);
    }

    @Safe
    public static int balanceOf(Hash160 owner) throws Exception {
        if (!Hash160.isValid(owner)) {
            throw new Exception("The parameter 'owner' must be a 20-byte address.");
        }
        return getBalance(owner);
    }

    @Safe
    public static Iterator<ByteString> tokensOf(Hash160 owner) throws Exception {
        if (!Hash160.isValid(owner)) {
            throw new Exception("The parameter 'owner' must be a 20-byte address.");
        }
        return (Iterator<ByteString>) Storage.find(Storage.getReadOnlyContext(), createTokensOfPrefix(owner),
                FindOptions.ValuesOnly);
    }

    public static boolean transfer(Hash160 to, ByteString tokenId, Object data) throws Exception {
        if (!Hash160.isValid(to)) {
            throw new Exception("The parameter 'to' must be a 20-byte address.");
        }
        if (tokenId.length() > 64) {
            throw new Exception("The parameter 'tokenId' must be a valid NFT ID (64 or less bytes long).");
        }
        Hash160 owner = ownerOf(tokenId);
        if (!Runtime.checkWitness(owner)) {
            return false;
        }
        if (owner != to) {
            ownerOfMap.put(tokenId, to.toByteArray());

            new StorageMap(ctx, createTokensOfPrefix(owner)).delete(tokenId);
            new StorageMap(ctx, createTokensOfPrefix(to)).put(tokenId, 1);

            decreaseBalanceByOne(owner);
            increaseBalanceByOne(to);
        }
        onTransfer.fire(owner, to, 1, tokenId);
        if (new ContractManagement().getContract(to) != null) {
            Contract.call(to, "onNEP11Payment", CallFlags.All, new Object[]{owner, 1, tokenId, data});
        }
        return true;
    }
    // endregion NEP-11 methods
    // region non-divisible NEP-11 methods

    @Safe
    public static Hash160 ownerOf(ByteString tokenId) throws Exception {
        if (tokenId.length() > 64) {
            throw new Exception("The parameter 'tokenId' must be a valid NFT ID (64 or less bytes long).");
        }
        ByteString owner = ownerOfMap.get(tokenId);
        if (owner == null) {
            throw new Exception("This token id does not exist.");
        }
        return new Hash160(owner);
    }

    // endregion non-divisible NEP-11 methods
    // region optional NEP-11 methods

    @Safe
    public static Iterator<ByteString> tokens() {
        return (Iterator<ByteString>) registryMap.find(FindOptions.ValuesOnly);
    }

    @Safe
    public static Map<String, String> properties(ByteString tokenId) throws Exception {
        if (tokenId.length() > 64) {
            throw new Exception("The parameter 'tokenId' must be a valid NFT ID (64 or less bytes long).");
        }
        ByteString token = registryMap.get(tokenId);
        if (token == null) {
            throw new Exception("This token id does not exist.");
        }
        TokenState tokenState = (TokenState) new StdLib().deserialize(token);
        Map<String, String> p = new Map<>();
        p.put("name", tokenState.name);
        p.put("image", tokenState.image);
        p.put("desc", tokenState.desc);
        return p;
    }


    // endregion optional NEP-11 methods
    // region events

    @DisplayName("Transfer")
    private static Event4Args<Hash160, Hash160, Integer, ByteString> onTransfer;

    /**
     * This event is intended to be fired before aborting the VM. The first argument should be a message and the
     * second argument should be the method name whithin which it has been fired.
     */
    @DisplayName("Error")
    private static Event2Args<String, String> error;
    // region custom methods

    @Safe
    public static Hash160 contractOwner() {
        return Storage.getHash160(Storage.getReadOnlyContext(), contractOwnerKey);
    }

    public static void mint(Hash160 owner, TokenState tokenState) {
        if (!Runtime.checkWitness(contractOwner())) {
            fireErrorAndAbort("No authorization.", "mint");
        }
        Integer tokenNO = totalSupply() + 1;
        ByteString tokenId = new ByteString(tokenNO);

        registryMap.put(tokenId, new StdLib().serialize(tokenState));

        ownerOfMap.put(tokenId, owner.toByteArray());

        new StorageMap(ctx, tokensOfKey).put(owner.toByteArray(), tokenId);

        increaseBalanceByOne(owner);
        incrementTotalSupplyByOne();
        onTransfer.fire(null, owner, 1, tokenId);
    }

    @Struct
    static class TokenState {
        String name;
        String image;
        String desc;
    }
    // region private helper methods

    private static int getBalance(Hash160 owner) {
        return balanceMap.getIntOrZero(owner.toByteArray());
    }

    private static void fireErrorAndAbort(String msg, String method) {
        error.fire(msg, method);
        Helper.abort();
    }

    private static void increaseBalanceByOne(Hash160 owner) {
        balanceMap.put(owner.toByteArray(), getBalance(owner) + 1);
    }

    private static void decreaseBalanceByOne(Hash160 owner) {
        balanceMap.put(owner.toByteArray(), getBalance(owner) - 1);
    }

    private static void incrementTotalSupplyByOne() {
        Storage.put(Storage.getStorageContext(), totalSupplyKey, totalSupply() + 1);
    }

    private static void decrementTotalSupplyByOne() {
        Storage.put(Storage.getStorageContext(), totalSupplyKey, totalSupply() - 1);
    }

    private static byte[] createTokensOfPrefix(Hash160 owner) {
        return Helper.concat(tokensOfKey, owner.toByteArray());
    }

    // endregion private helper methods

}

```

## 代码分析

### Import
引用了neow3j devpack包中的类,这些类的详情可以在[neow3j devpack](https://javadoc.io/doc/io.neow3j/devpack/latest/index.html)的javadoc中查看
```java
import io.neow3j.devpack.*;
import io.neow3j.devpack.Runtime;
import io.neow3j.devpack.annotations.*;
import io.neow3j.devpack.constants.CallFlags;
import io.neow3j.devpack.constants.FindOptions;
import io.neow3j.devpack.constants.NativeContract;
import io.neow3j.devpack.constants.NeoStandard;
import io.neow3j.devpack.contracts.ContractManagement;
import io.neow3j.devpack.contracts.StdLib;
import io.neow3j.devpack.events.Event2Args;
import io.neow3j.devpack.events.Event3Args;
import io.neow3j.devpack.events.Event4Args;
```

### 头注解
```java
//你合约Token的名字,如果设置默认使用类的名字
@DisplayName("StampToken")
//@ManifestExtra.ManifestExtras({@ManifestExtra(key = "author", value = "jackcao"),@ManifestExtra(key = "email", value = "cjcjcjcjcj9684@163.com")})
//额外说明信息这些信息会在manifest中看到
@ManifestExtra(key = "author", value = "jackcao")
@ManifestExtra(key = "email", value = "cjcjcjcjcj9684@163.com")
//manifest中的supportedStandards属性,代表遵循NEP11规范
@SupportedStandard(neoStandard = NeoStandard.NEP_11)
//设置你的合约允许去调用哪些合约
@Permission(nativeContract = NativeContract.ContractManagement)
```
### 合约属性和存储属性
```java
static final StorageContext ctx = Storage.getStorageContext();
//存储totalSupply
static final StorageMap contractMap = new StorageMap(ctx, 0);
//存储已发行的NFT的tokenid
static final StorageMap registryMap = new StorageMap(ctx, 1);
//存储tokenid的归属关系
static final StorageMap ownerOfMap = new StorageMap(ctx, 2);
//存储每个用户拥有的NFT个数
static final StorageMap balanceMap = new StorageMap(ctx, 3);
//存储合约的所有者，只有合约的所有者才能操作一些特殊功能，比如发行制作NFT
static final StorageMap ContractMetadata =  new StorageMap(ctx, 4);
// region keys of key-value pairs in NFT properties
//NFT的三个属性，名称，描述，图片
static final String propName = "name";
static final String propDescription = "description";
static final String propImage = "image";

static final StorageMap propNameMap = new StorageMap(ctx, 8);
static final StorageMap propDescriptionMap = new StorageMap(ctx, 9);
static final StorageMap propImageMap = new StorageMap(ctx, 10);

static final byte[] totalSupplyKey = new byte[]{0x10};
static final byte[] tokensOfKey = new byte[]{0x11};
```
- 所有的变量和方法都必须是`static`,静态变量使用`final`,如果需要可变的变量，就放到`Storage`里面去操作。
- 合约属性中定义的这些属性通常是一些常量，可以在智能合约方法的内部使用，当智能合约在任意实例上运行时，这些属性的值都保持不变。
- 存储属性:在开发智能合约时，必须将应用程序的数据存储在区块链上。当创建一个智能合约或者交易使用这个合约时，合约的代码需要读写它的存储空间。存储在智能合约存储区中的所有数据在智能合约的调用期间会自动持久化。区块链中的全节点会存储链上每一个智能合约的状态。
Neo 提供了基于键值对的数据访问接口。可以使用键从智能合约中读取、删除数据或将数据记录写入到智能合约中.智能合约可以使用 `StorageMap` 类来读写持久性存储区，`StorageMap`你可以当做是个`Map`来进行操作

### Deploy Update  Destroy
```java
// region deploy, update, destroy

@OnDeployment
public static void deploy(Object data, boolean update) {
    Transaction Tx = (Transaction) Runtime.getScriptContainer();
    if (!update) {
        ContractMetadata.put("Owner", Tx.sender.toByteString());
        contractMap.put(totalSupplyKey, 0);
    }
}

```
- 当合约部署到区块链上之后，`deploy()`方法就会被触发，对应的java代码中`@OnDeployment`注解的方法。
- -  我们在`deploy`方法中设置了`Owner`，为合约部署者，后面代码中可以为`Owner`账户提供更多的功能，如只有是`Owner`才能调用`mint`方法。
- -  设置了`totalSupplyKey` 总量初始值为0

```java
public static void update(ByteString script, String manifest) throws Exception {
    ByteString owner = ContractMetadata.get("Owner");
    Hash160 owner160 = new Hash160(owner);
    if (!Runtime.checkWitness(owner160)) {
//            error.fire("No authorization", "update");
//            Helper.abort();
        throw new Exception("No authorization.update");
    }
    ContractManagement.update(script, manifest);
}

```
- 用户触发`update`方法，先进行`Owner`判断，如果是合约所有者，就调用`ContractManagement.update`最后还会调用`deploy()`。

```java
public static void destroy() throws Exception {
    ByteString owner = ContractMetadata.get("Owner");
    Hash160 owner160 = new Hash160(owner);
    if (!Runtime.checkWitness(owner160)) {
//            error.fire("No authorization", "destroy");
//            Helper.abort();
        throw new Exception("No authorization.destroy");
    }
    ContractManagement.destroy();
}
```
- `destroy()`会将合约的所有数据全部清除，合约也不能再使用。

## NEP-11 规范中的方法
如果方法是读取数据，不涉及数据的改变，可以使用`@Safe`注解
```java
@Safe
public static String symbol() {
    return "STAMP";
}

@Safe
public static int decimals() {
    return 0;
}

@Safe
public static int totalSupply() {
    return Storage.getInt(Storage.getReadOnlyContext(), totalSupplyKey);
}

@Safe
public static int balanceOf(Hash160 owner) throws Exception {
    if (!Hash160.isValid(owner)) {
        throw new Exception("The parameter 'owner' must be a 20-byte address.");
    }
    return getBalance(owner);
}

@Safe
public static Iterator<ByteString> tokensOf(Hash160 owner) throws Exception {
    if (!Hash160.isValid(owner)) {
        throw new Exception("The parameter 'owner' must be a 20-byte address.");
    }
    return (Iterator<ByteString>) Storage.find(ctx.asReadOnly(), createTokensOfPrefix(owner),
            (byte) (FindOptions.KeysOnly | FindOptions.RemovePrefix));
}

public static boolean transfer(Hash160 to, ByteString tokenId, Object data) throws Exception {
    if (!Hash160.isValid(to)) {
        throw new Exception("The parameter 'to' must be a 20-byte address.");
    }
    if (tokenId.length() > 64) {
        throw new Exception("The parameter 'tokenId' must be a valid NFT ID (64 or less bytes long).");
    }
    Hash160 owner = ownerOf(tokenId);
    if (!Runtime.checkWitness(owner)) {
        return false;
    }
    onTransfer.fire(owner, to, 1, tokenId);
    if (owner != to) {
        ownerOfMap.put(tokenId, to.toByteArray());

        new StorageMap(ctx, createTokensOfPrefix(owner)).delete(tokenId);
        new StorageMap(ctx, createTokensOfPrefix(to)).put(tokenId, 1);

        decreaseBalanceByOne(owner);
        increaseBalanceByOne(to);
    }
    if (ContractManagement.getContract(to) != null) {
        Contract.call(to, "onNEP11Payment", CallFlags.All, new Object[]{owner, 1, tokenId, data});
    }
    return true;
}
```
## Non-divisible NFT 方法 ownerOf
根据TokenId,去查询NFT的所有者是谁
```java
@Safe
    public static Hash160 ownerOf(ByteString tokenId) throws Exception {
        if (tokenId.length() > 64) {
            throw new Exception("The parameter 'tokenId' must be a valid NFT ID (64 or less bytes long).");
        }
        ByteString owner = ownerOfMap.get(tokenId);
        if (owner == null) {
            throw new Exception("This token id does not exist.");
        }
        return new Hash160(owner);
    }
```
## 规范中的可选方法，对于我们来说是必须要用的方法
查询所有的Token。
```java
 @Safe
    public static Iterator<Iterator.Struct<ByteString, ByteString>> tokens() {
        return (Iterator<Iterator.Struct<ByteString, ByteString>>) registryMap.find(FindOptions.RemovePrefix);
    }
```
根据tokenId来查询NFT的详情,将属性从不同的`Stroage`里取出，放入`Map`中返回。
```java
    @Safe
    public static Map<String, String> properties(ByteString tokenId) throws Exception {
        if (tokenId.length() > 64) {
            throw new Exception("The parameter 'tokenId' must be a valid NFT ID (64 or less bytes long).");
        }
        Map<String, String> p = new Map<>();
        ByteString tokenName = propNameMap.get(tokenId);
        if (tokenName == null) {
            throw new Exception("This token id does not exist.");
        }

        p.put(propName, tokenName.toString());
        ByteString tokenDescription = propDescriptionMap.get(tokenId);
        if (tokenDescription != null) {
            p.put(propDescription, tokenDescription.toString());
        }
        ByteString tokenImage = propImageMap.get(tokenId);
        if (tokenImage != null) {
            p.put(propImage, tokenImage.toString());
        }
        return p;
    }
```
## 事件
事件可以理解把一些自己定义的日志记录到区块中，来方便查询。事件通过以下方法进行定义
```java
@DisplayName("Transfer")
private static Event4Args<Hash160, Hash160, Integer, ByteString> onTransfer;

@DisplayName("Mint")
private static Event3Args<Hash160, ByteString, Map<String, String>> onMint;

@DisplayName("MintNeoLine")
private static Event3Args<Hash160, ByteString, MyStruct> onMintNeoLine;
```
通过`onTransfer.fire(owner, to, 1, tokenId);` `onMint.fire(owner, tokenId, properties);` `变量名.fire(xxxx)`传入参数来触发

在区块中查询触发的事件：
```bash
Eventname:
Mint
VM State:
HALT
Contract:
0x8bcccdceff8b30361fb50a76854bf52cf0102e0f
State:
ByteString: 0x40051caf48052ef3bab78b7796cf799e2c5c32f7
ByteString: 01
Map: {"key":{"type":"ByteString","value":"ZGVzY3JpcHRpb24="},"value":{"type":"ByteString","value":"WkdWelkzSnBjSFJwYjI0Z2JYa2dabWx5YzNRZ2MzUmhiWEE9"}},{"key":{"type":"ByteString","value":"bmFtZQ=="},"value":{"type":"ByteString","value":"YzNSaGJYQXg="}},{"key":{"type":"ByteString","value":"aW1hZ2U="},"value":{"type":"ByteString","value":"YUhSMGNEb3ZMekV5Tnk0d0xqQXVNUT09"}}
```
## 自定义方法
除了规范中要求实现的方法外，还可以根据自己的需要进行添加
`contractOwner`查询当前的合约所有者
```java
@Safe
public static Hash160 contractOwner() {
    ByteString owner = ContractMetadata.get("Owner");
    Hash160 owner160 = new Hash160(owner);
    return owner160;
}
```
`mint`，`mintNeoLine`，`mintNeoLineStr`这3个方法，都是用来铸造一个NFT的，演示了3种不同参数的方法实现。
接收参数，判断操作者是否是合约所有者，如果不是就停止执行。如果是合约所有者，将传递的属性放入不同的`StorageMap`中。更新关联的数据，最后触发事件。
```java
public static void mint(Hash160 owner,Map<String, String> properties) {
    Integer tokenNO = totalSupply() + 1;
    ByteString tokenId = new ByteString(tokenNO);
    if (!Runtime.checkWitness(contractOwner())) {
        fireErrorAndAbort("No authorization.", "mint");
    }
    if (!properties.containsKey(propName)) {
        fireErrorAndAbort("The properties must contain a value for the key 'name'.", "mint");
    }
    String tokenName = properties.get(propName);
    propNameMap.put(tokenId, tokenName);
    if (properties.containsKey(propDescription)) {
        String description = properties.get(propDescription);
        propDescriptionMap.put(tokenId, description);
    }
    if (properties.containsKey(propImage)) {
        String image = properties.get(propImage);
        propImageMap.put(tokenId, image);
    }

    registryMap.put(tokenId, tokenId);
    ownerOfMap.put(tokenId, owner.toByteArray());
    new StorageMap(ctx, createTokensOfPrefix(owner)).put(tokenId, 1);

    increaseBalanceByOne(owner);
    incrementTotalSupplyByOne();
    onMint.fire(owner, tokenId, properties);
}

@Struct
static class MyStruct {
    String name;
    String image;
    String description;
}


public static void mintNeoLine(Hash160 owner,Object data) {
    MyStruct myData = (MyStruct) data;
    Integer tokenNO = totalSupply() + 1;
    ByteString tokenId = new ByteString(tokenNO);
    if (!Runtime.checkWitness(contractOwner())) {
        fireErrorAndAbort("No authorization.", "mint");
    }
    propNameMap.put(tokenId, myData.name);
    propDescriptionMap.put(tokenId, myData.description);
    propImageMap.put(tokenId, myData.image);

    registryMap.put(tokenId, tokenId);
    ownerOfMap.put(tokenId, owner.toByteArray());
    new StorageMap(ctx, createTokensOfPrefix(owner)).put(tokenId, 1);

    increaseBalanceByOne(owner);
    incrementTotalSupplyByOne();
    onMintNeoLine.fire(owner, tokenId, myData);
}

public static void mintNeoLineStr(Hash160 owner,String name,String image, String description) {
    Integer tokenNO = totalSupply() + 1;
    ByteString tokenId = new ByteString(tokenNO);
    if (!Runtime.checkWitness(contractOwner())) {
        fireErrorAndAbort("No authorization.", "mint");
    }
    propNameMap.put(tokenId, name);
    propDescriptionMap.put(tokenId, description);
    propImageMap.put(tokenId, image);

    registryMap.put(tokenId, tokenId);
    ownerOfMap.put(tokenId, owner.toByteArray());
    new StorageMap(ctx, createTokensOfPrefix(owner)).put(tokenId, 1);

    increaseBalanceByOne(owner);
    incrementTotalSupplyByOne();
    MyStruct myData = new MyStruct();
    myData.image=image;
    myData.name=name;
    myData.description=description;
    onMintNeoLine.fire(owner, tokenId, myData);
}
```
## 辅助方法
```java
private static int getBalance(Hash160 owner) {
        return balanceMap.getIntOrZero(owner.toByteArray());
    }

private static void fireErrorAndAbort(String msg, String method) {
    error.fire(msg, method);
    Helper.abort();
}

private static void increaseBalanceByOne(Hash160 owner) {
    balanceMap.put(owner.toByteArray(), getBalance(owner) + 1);
}

private static void decreaseBalanceByOne(Hash160 owner) {
    balanceMap.put(owner.toByteArray(), getBalance(owner) - 1);
}

private static void incrementTotalSupplyByOne() {
    int updatedTotalSupply = contractMap.getInt(totalSupplyKey) + 1;
    contractMap.put(totalSupplyKey, updatedTotalSupply);
}

private static void decrementTotalSupplyByOne() {
    int updatedTotalSupply = contractMap.getInt(totalSupplyKey) - 1;
    contractMap.put(totalSupplyKey, updatedTotalSupply);
}

private static byte[] createTokensOfPrefix(Hash160 owner) {
    return Helper.concat(tokensOfKey, owner.toByteArray());
}
```