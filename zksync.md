操作系统：MAC OS 10.14.x
zksync：https://github.com/matter-labs/zksync，master 分支
commit 899ca93052420c33583755cd2162143f461f36b3 

需要注意的是，阅读完整源代码需要查看dev分支，master分支是提供编译后的各种执行程序。在以下安装时，请科学上网。

## 环境准备 [zksync](https://github.com/matter-labs/zksync)/[docs](https://github.com/matter-labs/zksync/tree/master/docs)/**setup-dev.md**  

### 1、安装docker
```brew cask install docker```
### 2、安装Node & Yarn
```brew install nodejs yarn```
### 3、Axel
```brew install axel```
axel版本有差异，请采用axel 2.17.X，ubuntu、centos7、8环境，axel2.4、2.5版本均有问题，主要体现在执行：cargo install diesel_cli --no-default-features --features postgres无法通过。
### 4、Rust
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustc --version
rustc 1.46.0 (04488afe3 2020-08-24)
```
### 5、postgresql
```brew install postgresql```
### 6、Diesel cli
```cargo install diesel_cli --no-default-features --features postgres```
过程比较慢，耐心等待。。。
### 7、sqlx cli
```cargo install sqlx-cli```
### 8、solc
必须是v0.5.X
```brew install solidity@5```
### 9、下载zksync
```git clone https://github.com/matter-labs/zksync.git```
### 10、编辑~/. bash_profile:
```
export ZKSYNC_HOME=/path/to/zksync
export PATH=$ZKSYNC_HOME/bin:$PATH
```
到这里，环境依赖已经完成了

##  启动zksync    [zksync](https://github.com/matter-labs/zksync)/[docs](https://github.com/matter-labs/zksync/tree/master/docs)/**launch.md**

### 1、设置本地运行环境
```
cd zksync/bin && ./zk     #installs and builds zk itself
zk init                   #时间会比较久，耐心等待需要下载约8G的电路设置文件
```
初始化只需要一次即可，查看``infrastructure/zk/src/init.ts``
```
export async function init() {
    if (!process.env.CI) {
        await checkEnv();
        await env.gitHooks();
        await up();
    }
    await utils.allowFail(run.yarn());
    await run.plonkSetup();
    await run.verifyKeys.unpack();
    await db.setup();
    await contract.buildDev();
    await run.deployERC20('dev');
    await contract.build();
    await server.genesis();
    await contract.redeploy();
}

```
### 2、启动容器
```zk up```
查看``infrastructure/zk/src/up.ts``,该命令启动了geth 1，postgres，dev-ticker，tesseracts
```
export async function up() {
    await utils.spawn('docker-compose up -d postgres geth dev-ticker');
    await utils.spawn('docker-compose up -d tesseracts');
}
```
退出容器
```zk down```

### 3、运行zksync server
```
$ zk server
    Finished release [optimized] target(s) in 1.28s
     Running `target/release/zksync_server`
[2020-12-07T06:23:10Z INFO  zksync_server] Running the zkSync server
[2020-12-07T06:23:10Z INFO  zksync_server] Starting the Core actors
[2020-12-07T06:23:10Z INFO  zksync_core::state_keeper] Loaded committed state: last block number: 8, unprocessed priority op: 4
[2020-12-07T06:23:10Z INFO  zksync_core::state_keeper] created state keeper, root hash = Fr(0x2e1da2ec83289a3bb50282b56dee671ff815ac520f5a64c2ce89f3e62284a7dc)
[2020-12-07T06:23:10Z INFO  zksync_core::state_keeper] Executed restored proposed block: 0 transactions, 0 priority operations, 0 failed transactions
[2020-12-07T06:23:10Z INFO  zksync_server] Starting the API server actors
[2020-12-07T06:23:10Z INFO  zksync_server] Starting the Ethereum sender actors
[2020-12-07T06:23:10Z INFO  zksync_server] Starting the Prover server actors
[2020-12-07T06:23:10Z INFO  zksync_witness_generator] Starting witness generator (1,2)
[2020-12-07T06:23:10Z INFO  zksync_witness_generator::witness_generator] preparing prover data routine started with start_block(1), block_step(2)
[2020-12-07T06:23:10Z INFO  zksync_witness_generator] Starting witness generator (2,2)
[2020-12-07T06:23:10Z INFO  zksync_witness_generator::witness_generator] preparing prover data routine started with start_block(2), block_step(2)
[2020-12-07T06:23:10Z INFO  zksync_core::mempool] 0 transactions were restored from the persistent mempool storage

```
第一次运行会编译rust，时间比较久，请耐心等待
查看``infrastructure/zk/src/server.ts``，用ts来编译、运行了``core/bin/server/src/main.rs``。
```
export async function server() {
    await utils.spawn('cargo run --bin zksync_server --release');
}
```

##### 关于etc/env/dev.env
这个文件是在init的时候从dev.env.example复制生成的。可以修改其中的参数，
如端口参数、出块间隔、见证者数量等。在本地的部署测试中，发现``zk server``的时候端口被占用的错误，首先请排查端口占用情况：``lsof -i:8545``,如果没有占用情况，那可以修改dev.env的端口在尝试一下。有需要的话，需要同步修改``etc/js/env-config.js``的端口。
### 4、运行zksync cli
```
cd bin && vim zcli      #拷贝zk文件内容，作出相应修改
```
zcli文件内容：
```
#!/bin/bash

if [ -z "$1" ]; then
    cd $ZKSYNC_HOME
    yarn && yarn zcli build
else
    # can't start this with yarn since it has quirks with `--` as an argument
    node -- $ZKSYNC_HOME/infrastructure/zcli/build/index.js "$@"
fi
```
```
$ ./zcli      #build xcli

$ zcli -h   #查看命令行帮助
Usage: zcli [options] [command]

Options:
  -V, --version                                    output the version number
  -n, --network <network>                          select network (default: "localhost")
  -h, --help                                       display help for command

Commands:
  account [address]                                view account info
  transaction <tx_hash>                            view transaction info
  transfer [options] [amount] [token] [recipient]  make a transfer
  deposit [options] [amount] [token] [recipient]   make a deposit
  await [options] <type> <tx_hash>                 await for transaction commitment/verification
  networks                                         view configured networks
  wallets                                          view saved wallets
  help [command]                                   display help for command
```
添加账号：
```
$ zcli wallets add 0x27593fea79697e947890ecbecce7901b0008345e5d7259710d0dd5e500d040be. 
[WARNING]: private keys are stored unencrypted
"0xde03a0B5963f75f1C8485B355fF6D30f3093BDE7"

$ zcli wallets default 0xde03a0b5963f75f1c8485b355ff6d30f3093bde7
"0xde03a0b5963f75f1c8485b355ff6d30f3093bde7"

$ zcli wallets 
[
    "0xde03a0b5963f75f1c8485b355ff6d30f3093bde7",
    "0xab9fc101e0958669c92d71855f41aa5c949e5d8e",
    "0xdb850fd6dd80f0f689f2f4671a003c99ed73cd49"
]

$ zcli account 0xde03a0b5963f75f1c8485b355ff6d30f3093bde7
{
    "address": "0xde03a0b5963f75f1c8485b355ff6d30f3093bde7",
    "network": "localhost",
    "account_id": 0,
    "nonce": 0,
    "balances": {
        "ETH": "0.0004132"
    }
}

$ zcli deposit 1000 ETH 0x52312AD6f01657413b2eaE9287f6B9ADaD93D5FE --fast
"0x39e8a769a1be699c01afad650f3d5ff3c570bb805a9c7b23177080cff35c8ae5"

$ zcli transaction 0x39e8a769a1be699c01afad650f3d5ff3c570bb805a9c7b23177080cff35c8ae5
{
    "network": "localhost",
    "transaction": {
        "status": "success",
        "from": "0xde03a0b5963f75f1c8485b355ff6d30f3093bde7",
        "to": "0x52312ad6f01657413b2eae9287f6b9adad93d5fe",
        "hash": "0x39e8a769a1be699c01afad650f3d5ff3c570bb805a9c7b23177080cff35c8ae5",
        "operation": "Deposit",
        "nonce": -1,
        "amount": "1000.0",
        "token": "ETH"
    }
}

$ zcli account 0x52312AD6f01657413b2eaE9287f6B9ADaD93D5FE
{
    "address": "0x52312AD6f01657413b2eaE9287f6B9ADaD93D5FE",
    "network": "localhost",
    "account_id": 1,
    "nonce": 5,
    "balances": {
        "ETH": "2994.9995868"
    }
} #原先有抵押过了

$ zcli transfer 2 ETH 0xdB850fD6DD80f0F689F2f4671A003c99ed73cD49

...

```
``0xde03a0B5963f75f1C8485B355fF6D30f3093BDE7 ``为dev.env文件中有L2给 geth1 发交易的地址，私钥也在配置文件中。

大家可以按照提示使用zcli来向zksync发送交易，查询交易等操作。

更多关于zksync、零知识证明等的精彩分享，请加微信cw_walker，注明简书，我们有一群爱好区块链的创客，可以一起学习成长！