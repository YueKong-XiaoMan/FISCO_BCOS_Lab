# 我的第一个区块链网络：FISCO BCOS 2.0 搭建与重建全流程实录

> 本文记录了我从零开始在阿里云上搭建第一个区块链网络（FISCO BCOS 2.0）的完整过程。  
> 从初次部署（忘记大部分的截过程图了，但是仍会写清楚一步一步的操作，并写出遇到的困难和解决方法）到重建环境，经历了多次出错与修复，最终成功实现智能合约部署与调用。

---

## 一、前言：为什么要自己搭区块链

  在学习区块链的过程中，老师希望我们不仅停留在概念层面，而是能够**通过实际操作深入理解区块链的底层运行机制**。  因此,尝试**亲手搭建一个能够真实运行的区块链网络**。在平台选择上，使用了国产开源框架 **FISCO BCOS 2.0** —— 它由国内团队开发，文档完善、社区活跃，并且支持智能合约的编写与部署，非常适合教学与实践。  为了方便部署与环境管理，我选择在 **阿里云 ECS（Ubuntu 系统）** 上进行搭建，这是依据了**FISCO BCOS 文档**中硬件和系统要求。这样不仅便于远程操作，也能在遇到问题时快速重置或重新配置环境。

目标很简单：
- 成功搭建一个四节点区块链网络；
- 在控制台上部署并调用 “HelloWorld” 合约；
- 理解整个流程的逻辑与结构。

---

## 二、环境与工具准备

### 1. 云服务器环境

| 项目     | 配置                |
| ------ | ----------------- |
| 云服务商   | 阿里云 ECS           |
| 系统版本   | Ubuntu 22.04 LTS  |
| CPU/内存 | 4 核 / 8GB         |
| 用户     | root              |

### 2. 主要文件与目录规划

| 目录 | 作用 |
|------|------|
| `/root/fisco` | 主目录 |
| `nodes/` | 区块链节点文件夹 |
| `console/` | 控制台工具目录 |
| `contracts/` | 智能合约文件存放处 |
| `scripts/` | 各类运行脚本 |

---

## 三、第一次搭建区块链网络

### 1.安装ubuntu依赖
```bash
sudo apt install -y openssl curl
```
### 2. 下载安装脚本
```bash
## 创建操作目录
cd ~ && mkdir -p fisco && cd fisco
## 下载脚本
curl -#LO https://github.com/FISCO-BCOS/FISCO-BCOS/releases/download/v2.11.0/build_chain.sh && chmod u+x build_chain.sh
```
#### 💡 关于 “下载安装脚本” 容易出问题的说明
```bash
curl -#LO https://github.com/FISCO-BCOS/FISCO-BCOS/releases/download/v2.11.0/build_chain.sh && chmod u+x build_chain.sh
```
会遇到下载失败或卡住报错的情况，这可能是因为：
  ⚠️ 
1. **GitHub 连接太慢或超时**  
    阿里云 ECS 在国内访问 GitHub 往往速度很慢，甚至直接断开连接。
    
2. **curl 工具版本或参数问题**  
    某些 Ubuntu 镜像自带的 `curl` 不支持 `-#LO` 参数，执行时会报错。
    
3. **网络防火墙或代理干扰**  
    如果你的云服务器网络策略比较严格（例如企业校园网），可能阻止了外部下载。

当我在第一次搭建过程中，出现这个问题，一开始选择了镜像地址，但还是下载失败并且下载极慢，通过了解发现镜像并不总是同步或者可用的，它只是中转了GitHub内容的服务。这些镜像有时会，没有同步最新的 release、临时宕机或被防火墙拦截、限速或返回不完整的文件等情况。最后导致了执行时报错。还有一种情况，服务器网络出口被限制，因为阿里云，腾讯云等服务器有时会默认不开放访问部分境外地址，即使我们换了镜像，也可能走的还是相同的出口 IP，从而被限制。当然还有很多情况导致镜像下载的失败，也没有深入探索。
最后我成功下载，是采用了本地下载加上传。这时候已经在服务器中输入下载命令，当下载开始后按下 `Ctrl + C` 暂停进程，然后按住 `Ctrl` 点击命令中的下载链接，在本地电脑浏览器中打开并完成下载。这样不仅速度更快，也能避免服务器网络不稳定的问题。下载完成后，我使用 **XFTP** 工具，将下载好的 `.tar.gz` 文件上传到服务器的工作目录（如 `~/fisco`），并赋予执行权限。这种方法虽然多了一步，但整体过程更加可靠，也能有效避免因为控制台连接断开导致的下载失败。
### 3. 解压 FISCO BCOS 二进制包
当脚本文件准备好后，下一步是让 **FISCO BCOS** 的二进制程序就绪。  
二进制包包含区块链节点运行所需的核心组件，是后续搭建网络的重要基础。
  登录服务器后，在对应目录下执行以下命令进行解压：
```bash    
tar -zxvf fisco-bcos.tar.gz
```
4. 解压完成后，会出现 `fisco-bcos` 可执行文件。  
为确保能正常运行，请添加执行权限：
```bash
chmod u+x fisco-bcos
```
执行完上述命令后，二进制文件已准备就绪，正式进入网络搭建阶段。
### 4. 生成四节点区块链

```bash
bash build_chain.sh -l 127.0.0.1:4 -p 30300 -e ./fisco-bcos
```
![[Pasted image 20251029234732.png]]

### 5. 启动节点并验证状态

```bash
bash nodes/127.0.0.1/start_all.sh
```
![[Pasted image 20251030001141.png]]
### 6. 检查日志输出
  查看节点node0链接的节点数
```bash
tail -f nodes/127.0.0.1/node0/log/log*  | grep connected
```
 执行下面指令，检查是否在共识
```bash
tail -f nodes/127.0.0.1/node0/log/log*  | grep +++
```
这一步一般不会出现什么问题。这时候不断地输出信息，我们可以点ctrl+c暂停继续下一步。
![[Pasted image 20251030000215.png]]
## 四、配置及使用控制台

### 1. 准备依赖
安装java （推荐使用java 14）
```bash
# ubuntu系统安装java
sudo apt install -y default-jdk

#centos系统安装java
sudo yum install -y java java-devel
```
- 获取控制台并回到fisco目录
```bash
cd ~/fisco && curl -LO https://github.com/FISCO-BCOS/console/releases/download/v2.9.2/download_console.sh && bash download_console.sh
```
同样这里会出现下载脚本的情况，我们需要手动下载并解压。
- 拷贝控制台配置文件
```bash
# 最新版本控制台使用如下命令拷贝配置文件
cp -n console/conf/config-example.toml console/conf/config.toml
```
### 2. 启动并使用控制台

```bash
cd ~/fisco/console && bash start.sh
```
![[Pasted image 20251030002050.png]]
## 五、 部署及调用HelloWorld合约
HelloWorld合约提供两个接口，分别是`get()`和`set()`，用于获取/设置合约变量`name`。合约内容如下:
```bash
pragma solidity ^0.4.24;

contract HelloWorld {
    string name;

    function HelloWorld() {
        name = "Hello, World!";
    }

    function get()constant returns(string) {
        return name;
    }

    function set(string n) {
        name = n;
    }
}
```


### 1. 部署 HelloWorld 合约

```bash
[group:1]> deploy HelloWorld
```

HelloWorld合约已经内置于控制台中，位于控制台目录下`contracts/solidity/HelloWorld.sol`，参考下面命令部署即可。
```bash
# 在控制台输入以下指令 部署成功则返回合约地址
[group:1]> deploy HelloWorld
transaction hash: 0xd0305411e36d2ca9c1a4df93e761c820f0a464367b8feb9e3fa40b0f68eb23fa
contract address:0xb3c223fc0bf6646959f254ac4e4a7e355b50a344
```

### 2. 调用合约

```bash
# 查看当前块高
[group:1]> getBlockNumber
1

# 调用get接口获取name变量 此处的合约地址是deploy指令返回的地址
[group:1]> call HelloWorld 0xb3c223fc0bf6646959f254ac4e4a7e355b50a344 get
---------------------------------------------------------------------------------------------
Return code: 0
description: transaction executed successfully
Return message: Success
---------------------------------------------------------------------------------------------
Return values:
[
    "Hello,World!"
]
---------------------------------------------------------------------------------------------

# 查看当前块高，块高不变，因为get接口不更改账本状态
[group:1]> getBlockNumber
1

# 调用set设置name
[group:1]> call HelloWorld 0xb3c223fc0bf6646959f254ac4e4a7e355b50a344 set "Hello, FISCO BCOS"
transaction hash: 0x7e742c44091e0d6e4e1df666d957d123116622ab90b718699ce50f54ed791f6e
---------------------------------------------------------------------------------------------
transaction status: 0x0
description: transaction executed successfully
---------------------------------------------------------------------------------------------
Output
Receipt message: Success
Return message: Success
---------------------------------------------------------------------------------------------
Event logs
Event: {}

# 再次查看当前块高，块高增加表示已出块，账本状态已更改
[group:1]> getBlockNumber
2

# 调用get接口获取name变量，检查设置是否生效
[group:1]> call HelloWorld 0xb3c223fc0bf6646959f254ac4e4a7e355b50a344 get
---------------------------------------------------------------------------------------------
Return code: 0
description: transaction executed successfully
Return message: Success
---------------------------------------------------------------------------------------------
Return values:
[
    "Hello,FISCO BCOS"
]
---------------------------------------------------------------------------------------------

# 退出控制台
[group:1]> quit
```



以上是第一次搭建，除了网络问题，几乎在一个干净环境下的服务器都非常顺利。

---

## 六、重建环境：删除与重新部署

在第一次成功后，我决定重新练习一遍部署过程。  
但这一回遇到了不少麻烦，比如路径混乱、日志缺失、控制台无法退出等。
### 1. 删除旧节点
```bash
cd ~/fisco/nodes
bash stop_all.sh
rm -rf 127.0.0.1
ps -ef | grep fisco-bcos
```
### 2. 再次运行 build_chain.sh
![[Pasted image 20251030003212.png]]
![[Pasted image 20251030003617.png]]
脚本检测到 **`/root/fisco/nodes` 目录已经存在**，而脚本不会覆盖旧的节点目录，因此需要你**先删除旧的目录**再重新生成。
 **解决办法：**
在终端里输入以下命令删除旧目录，然后重新执行刚才的命令：
```bash
rm -rf nodes && bash build_chain.sh -l 127.0.0.1:4 -p 30300
```
💡 **说明：**
- `rm -rf` 是强制删除命令，请确保路径正确（这里是 `/root/fisco/nodes`）。
    
- 删除完成后可以用 `ls` 看看是否没有 `nodes` 文件夹了。
    
- 执行完毕后脚本会自动创建新的 4 个节点环境。
### 3.控制台异常与修复
#### 1.日志混乱
![[Pasted image 20251030004536.png]]
在这个截图上的情况其实 **不是错误**，而是因为之前创建的区块链网络 **被部分删除过日志目录**，导致它现在虽然能启动节点，但缺少日志文件路径。
报错：

`tail: cannot open 'nodes/127.0.0.1/node0/log/log*' for reading: No such file or directory`

这说明：  
`/root/fisco/nodes/127.0.0.1/node0/log/` 这个文件夹 **不存在**（被我之前删掉过）。
**解决方法（恢复日志文件夹）**
只要手动重新建一下日志目录就行：
```bash
mkdir -p /root/fisco/nodes/127.0.0.1/node0/log mkdir -p /root/fisco/nodes/127.0.0.1/node1/log mkdir -p /root/fisco/nodes/127.0.0.1/node2/log mkdir -p /root/fisco/nodes/127.0.0.1/node3/log
```
然后重新启动节点：
```bash
bash /root/fisco/nodes/127.0.0.1/stop_all.sh bash /root/fisco/nodes/127.0.0.1/start_all.sh
```
#### 2. 配置混乱
若发现节点配置文件（如 `config.toml`）内容异常、无法启动，可通过删除旧文件重新生成：
```bash
rm -f config.toml ./build_chain.sh -l 127.0.0.1:4 -p 30300 -e ./fisco-bcos
```
重新生成配置文件后，再次启动节点，通常即可恢复正常。

## 七、重建成功与验证

```bash
cd ~/fisco/console
bash start.sh
[group:1]> getBlockNumber
[group:1]> deploy HelloWorld
```
![[Pasted image 20251030150334.png]]

---

## 八、学习体会与总结

|模块|功能|
|:-:|:--|
|节点 (Node)|区块生成、共识执行|
|控制台 (Console)|用户与区块链交互的窗口|
|智能合约 (Solidity)|业务逻辑层|

---

### ✅ 收获

- 熟悉了 Linux 命令与路径管理；
    
- 理解了节点的配置与通信逻辑；
    
- 明白了出块与交易执行的关系；
    
- 学会了通过日志追踪问题来源，培养了独立排查的能力；
    
- 对 FISCO BCOS 的整体架构有了系统的理解，从底层网络到合约部署的每一步都有了实操经验。
    

---

### 🧩 问题与解决

- 通过多次环境搭建与重建，进一步理解了网络配置、权限设置和路径规范的重要性；
    
- 学会了利用日志文件分析错误来源，提高了问题定位与修复效率；
    
- 对下载失败、权限不足、配置混乱等常见问题形成了稳定的解决思路；
    
- 掌握了在不同终端和工具之间灵活切换、传输文件的技巧。
    

---

### 🚀 展望

- 计划进一步学习并验证 **共识机制** 的运行原理，深入理解 PBFT 的容错特性与节点协作过程；
    
- 尝试使用 **docker-compose** 进行自动化部署，提高实验效率；
    
- 学习更复杂的智能合约编写与调试方法；
    
- 探索区块链在业务层面的应用场景，如数据共享、身份认证等方向。
    

---

## 九、结束语

从最初部署到重建，我对区块链底层运行机制的理解逐渐深入。  
这篇文章不仅是一份部署记录，也是一份成长日志。

---

