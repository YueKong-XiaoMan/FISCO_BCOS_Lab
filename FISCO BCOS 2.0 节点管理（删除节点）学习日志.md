
## 一、实验概述

### 实验信息
**日期**: 2025年11月  
**实验目标**: 学习FISCO BCOS区块链网络的节点删除操作  
**实验环境**: 
- 云服务器: 阿里云ECS
- 操作系统: Ubuntu 22.04 64位
- 区块链框架: FISCO BCOS 2.0
- CPU/内存: 4 核 / 8GB     
- 网络规模: 4节点PBFT共识网络
### 实验背景
在此之前成功搭建了第一个区块链网络（FISCO BCOS 2.0），为了深入了解学习，老师也提出：在区块链运维过程中，节点的动态调整是常见需求。因此本次实验学习如何在PBFT共识机制下安全地删除节点，同时保持网络的稳定运行。
#### 为什么不能先进行增加节点的操作？
这主要是由**硬件资源限制**和**区块链节点的资源消耗特性**共同导致的。
   1.单节点资源消耗与服务器总资源的矛盾
    FISCO BCOS 等区块链节点运行时，会占用 CPU、内存、网络带宽等资源。单节点推荐配置通常为 **4 核 CPU、8GB 内存**（甚至更高），而我的的阿里云服务器是 “4核 8G”，仅能满足**单节点的基础资源需求**。 当运行 4 个节点时，资源已接近满载；若增加到 5 个节点，CPU 和内存会因过度分配而出现争抢，导致节点无法正常启动或运行不稳定。
   2.区块链节点的资源消耗特性
    节点不仅要处理交易、共识等核心逻辑，还需同步区块数据、维护 P2P 连接，这些操作会随节点数量增加**线性消耗资源**。在内存方面，每个节点需缓存区块、交易池等数据，5 个节点的内存总需求会超过 8GB；在CPU 方面，共识算法（如 PBFT、rPBFT）的计算开销随节点数增加而上升，4 核 CPU 无法支撑 5 个节点的并发计算。

## 二、 环境准备与验证

### 1. 服务器连接与目录确认
```bash
# 连接到阿里云ECS服务器
ssh root@8.148.85.144

# 进入FISCO BCOS工作目录
cd /root/fisco
```
**命令解释**:
- `ssh`: 安全Shell协议，用于远程连接服务器（这是一般在终端操作，可以通过xshell8等直接连接阿里云ECS服务器更加方便）
- `cd`: 改变当前工作目录

### 2. 网络结构确认
```bash
# 查看节点目录结构
ls -la nodes/127.0.0.1/
```
**预期输出**:
```
drwxr-xr-x 6 root root 4096 Nov  8 02:02 node0
drwxr-xr-x 6 root root 4096 Nov  8 02:02 node1  
drwxr-xr-x 6 root root 4096 Nov  8 02:02 node2
drwxr-xr-x 6 root root 4096 Nov  8 02:02 node3
```
**输出说明**: 显示4个节点目录，确认是4节点网络

### 3. 共识机制确认
```bash
# 检查节点配置中的共识设置
cat nodes/127.0.0.1/node0/config.ini | grep consensus
```
**可能结果**: 无输出（不同版本的 FISCO BCOS 配置文件结构可能有调整，也可能该配置项被封装在其他配置段中，导致通过`grep consensus`无法直接匹配到输出）或显示 `consensus_type=pbft`
![[Pasted image 20251108032031.png]]我的情况就是无输出状态，这可能是因为当前 FISCO BCOS 版本的共识配置被封装在其他模块，或采用了默认共识机制（如 PBFT）且未在`config.ini`中以`consensus`字段显式声明。不过并没有什么影响，只要我们是基于 PBFT 共识机制的逻辑（如遵循 “4 节点最大容错 1 个” 的规则）来删减节点，即使 `config.ini` 中无 `consensus` 显式输出，对节点删减操作**没有实质性影响**
**知识点**: 
- PBFT(Practical Byzantine Fault Tolerance): 实用拜占庭容错算法
- 要求: 至少需要 `2f+1` 个节点才能容忍f个故障节点
- 4节点网络: 最大容错f=1，删除1个节点后仍可正常工作

### 4. 当前网络状态检查
```bash
# 进入控制台目录
cd nodes/127.0.0.1/console

# 启动控制台
bash start.sh

# 查看当前共识节点列表
getSealerList
```
**预期输出**:
```json
{
    "051e4a13acb35ef7eba58605928eaf7501affd9e037e21743cb2c3d46e6dc1bac2b808abf7b53acf1c5f560ebd7d20040b51becdf46ecd6be699e6855a27fa45",
    "0e120722181662980c140af3519e6581bcd0b6aa8f228bd35f8849e44d2dea4ff50480f3d54be9a5186c3d54764b59e8fc420e19025fb77a29c234826235d95a", 
    "4c70a2c268b1f80662563b07476376e11a0786215871dfdb9c893b32f7169c5611ef97e4f142bb4353fee82477bc6d6609a9c7ba5da813d42d89f0cd6940c005",
    "4d12374a28a14694a58e1439e6c14a386737d4d87acb2e72f91bab7535271431be6a1551c099c71a0e0b46ce0b709451099c06a67e2fa267085bc52d596ee492"
}
```
**输出说明**: 显示4个节点ID，确认网络正常运行
![[Pasted image 20251108032501.png]]每个长字符串是节点的唯一标识（NodeID）。`getSealerList`用于查询当前参与共识（负责区块生成、验证）的节点集合，这些节点通过 PBFT 等共识算法协同工作，保障区块链的一致性和可靠性。我们看到的每个 NodeID 对应一个正在参与共识的节点，它们共同维护链的记账逻辑。
## 三、 节点删除详细操作流程

### 步骤1: 选择并停止目标节点

我们选择删除 `node3`，首先需要停止该节点进程：

```bash
# 进入node3目录
cd /root/fisco/nodes/127.0.0.1/node3

# 停止node3节点
./stop.sh
```
**命令输出**: `stop node3 success.`
![[Pasted image 20251108032828.png]]
**命令解释**:
- `./stop.sh`: 执行节点停止脚本，优雅关闭节点进程

### 步骤2: 验证节点已停止
```bash
# 检查node3进程是否仍在运行
ps aux | grep fisco-bcos | grep node3
```
**预期输出**: 无输出（表示节点已成功停止）

**命令解释**:
- `ps aux`: 显示所有运行中的进程
- `grep fisco-bcos`: 过滤出FISCO BCOS相关进程
- `grep node3`: 进一步过滤出node3的进程

### 步骤3: 获取节点身份标识

每个FISCO BCOS节点都有唯一的节点ID，删除操作需要此标识：

```bash
# 查看node3的节点ID
cat /root/fisco/nodes/127.0.0.1/node3/conf/node.nodeid
```
**预期输出**:
```
0e120722181662980c140af3519e6581bcd0b6aa8f228bd35f8849e44d2dea4ff50480f3d54be9a5186c3d54764b59e8fc420e19025fb77a29c234826235d95a
```
![[Pasted image 20251108033014.png]]
**文件说明**: 
- `node.nodeid`: 存储节点的唯一身份标识
- 格式: 128字符的十六进制字符串
### 步骤4: 通过控制台执行删除操作

#### 4.1 启动控制台
```bash
# 进入控制台目录
cd /root/fisco/nodes/127.0.0.1/console

# 启动控制台交互界面
bash start.sh
```
**成功标志**: 出现 `[group:1]>` 提示符

#### 4.2 执行节点删除命令
```bash
# 从共识节点列表中移除node3
removeNode 0e120722181662980c140af3519e6581bcd0b6aa8f228bd35f8849e44d2dea4ff50480f3d54be9a5186c3d54764b59e8fc420e19025fb77a29c234826235d95a
```
**预期输出**:
```json
{
    "code": 1,
    "msg": "Success"
}
```
![[Pasted image 20251108033155.png]]
**命令解释**:
- `removeNode`: FISCO BCOS系统管理合约的预编译命令
- 参数: 要删除的节点ID
- 返回值: `code=1` 表示操作成功

#### 4.3 验证删除结果
```bash
# 再次查看共识节点列表
getSealerList
```
**预期输出**:
```json
{
    "051e4a13acb35ef7eba58605928eaf7501affd9e037e21743cb2c3d46e6dc1bac2b808abf7b53acf1c5f560ebd7d20040b51becdf46ecd6be699e6855a27fa45",
    "4c70a2c268b1f80662563b07476376e11a0786215871dfdb9c893b32f7169c5611ef97e4f142bb4353fee82477bc6d6609a9c7ba5da813d42d89f0cd6940c005", 
    "4d12374a28a14694a58e1439e6c14a386737d4d87acb2e72f91bab7535271431be6a1551c099c71a0e0b46ce0b709451099c06a67e2fa267085bc52d596ee492"
}
```
![[Pasted image 20251108033211.png]]
**验证要点**: 
- 输出中只有3个节点ID
- node3的ID `0e120722...235d95a` 已从列表中消失

### 步骤5: 可选清理操作

如果需要完全移除节点，可以删除物理文件：

```bash
# 退出控制台
quit

# 删除node3目录
cd /root/fisco
rm -rf nodes/127.0.0.1/node3
```
**警告**: 此操作不可逆，确保节点已成功逻辑删除后再执行

## 四、故障排除与问题解决

### 问题1: 路径错误
**错误现象**:
```
cat: nodes/127.0.0.1/node0/config.ini: No such file or directory
```
**错误原因**: 在错误目录下执行命令  
**解决方案**: 确保在正确的根目录下操作
```bash
# 确认当前目录
pwd
# 应该显示: /root/fisco

# 如不在正确目录，切换过去
cd /root/fisco
```

### 问题2: 管理脚本缺失
**错误现象**:
```
bash: delete_node.sh: No such file or directory
```
**问题分析**: FISCO BCOS版本差异，缺少自动化管理脚本  
**解决方案**: 手动操作或使用替代方法

### 问题3: 控制台启动异常
**错误现象**: 没有出现 `[group:1]>` 提示符  
**排查步骤**:
```bash
# 检查控制台文件完整性
cd /root/fisco/nodes/127.0.0.1/console
ls -la

# 检查Java环境
java -version

# 尝试使用console.sh启动
bash console.sh
```

### 问题4: 节点ID格式错误
**错误现象**:
```json
{
    "msg": "Invalid node ID"
}
```
**错误原因**: 节点ID复制不完整或有字符错误  
**解决方案**: 重新从源文件复制完整节点ID
```bash
# 直接从文件获取，避免手动复制错误
cat /root/fisco/nodes/127.0.0.1/node3/conf/node.nodeid
```

##  五、操作结果验证

### 删除前后对比

| 项目   | 删除前     | 删除后      | 状态  |
| ---- | ------- | -------- | --- |
| 节点数量 | 4个      | 3个       | ✅   |
| 共识能力 | 容错f=1   | 容错f=1    | ✅   |
| 网络状态 | 正常运行    | 正常运行     | ✅   |
| 节点列表 | 包含node3 | 不包含node3 | ✅   |

### 网络健康检查
```bash
# 检查区块高度是否继续增长
getBlockNumber

# 检查节点连接状态
getGroupPeers

# 检查观察者节点
getObserverList
```
getBlockNumber：查询当前区块链的**区块高度**（已生成的区块总数）。通过多次执行可观察区块高度是否持续增长，判断区块链交易处理、区块生成是否正常。
getGroupPeers：查询当前节点所在**群组**内的节点连接状态，包含节点 ID、网络连接信息等，用于确认节点间 P2P 连接是否正常建立。
getObserverList：查询当前区块链网络中的**观察者节点列表**，观察者节点仅同步区块数据，不参与共识和记账，可用于确认其接入情况。

##  六、核心知识点总结

### PBFT共识机制深入理解
```python
# PBFT节点数量要求计算
def calculate_pbft_requirements(total_nodes):
    max_faulty = (total_nodes - 1) // 3
    min_required = 2 * max_faulty + 1
    return max_faulty, min_required

# 4节点网络
max_f, min_r = calculate_pbft_requirements(4)
print(f"4节点网络: 最大容错={max_f}, 最少需要={min_r}个节点")
# 输出: 4节点网络: 最大容错=1, 最少需要=3个节点

# 3节点网络  
max_f, min_r = calculate_pbft_requirements(3)
print(f"3节点网络: 最大容错={max_f}, 最少需要={min_r}个节点")
# 输出: 3节点网络: 最大容错=1, 最少需要=3个节点
```

### FISCO BCOS节点管理命令集

| 命令 | 功能 | 使用场景 |
|------|------|----------|
| `getSealerList` | 查看共识节点 | 网络状态检查 |
| `getObserverList` | 查看观察者节点 | 节点角色管理 |
| `removeNode` | 移除共识节点 | 节点删除操作 |
| `addObserver` | 添加观察者节点 | 节点角色调整 |
| `getGroupPeers` | 查看群组对等节点 | 网络连接状态 |

### 节点删除操作步骤

1. **物理层停止**: 停止节点进程，释放资源
2. **逻辑层移除**: 通过共识机制更新网络成员列表
3. **状态同步**: 其他节点同步新的网络配置
4. **资源清理**: 可选的文件清理操作（ ） 

## 六、 扩展实验建议

### 实验1: 节点添加操作（✔）
在FISCO BCOS 2.0 节点管理（增加节点）学习日志中写到
```bash
# 学习如何将新节点加入现有网络
addNode <new_node_id>
```

### 实验2: 观察者节点管理（✔）
这个仅仅是观察因此不单独做一个页面
```bash
# 将节点设置为观察者（不参与共识）
addObserver <node_id>

# 从观察者列表中移除
removeObserver <node_id>
```
addObserver <node_id>：将指定`node_id`的节点设置为**观察者节点**，该节点仅同步区块数据，不参与共识和记账。
removeObserver <node_id>：将指定`node_id`的节点从**观察者列表**中移除，使其不再以观察者身份参与网络。

### 实验3: 网络监控与维护
```bash
# 监控区块生成状态
getConsensusStatus

# 查看交易池状态
getTxPoolStatus

# 检查系统配置
getSystemConfigByKey
```

## 实验总结与反思

### 成功经验
1. ✅ 掌握了FISCO BCOS节点删除的完整流程
2. ✅ 理解了PBFT共识机制下的节点管理原理
3. ✅ 学会了使用控制台进行区块链网络运维
4. ✅ 积累了故障排查和问题解决的实践经验

### 注意事项
1. ⚠️ 删除节点前务必确认网络状态
2. ⚠️ 确保剩余节点数量满足共识要求
3. ⚠️ 操作前做好重要数据备份
4. ⚠️ 严格按照操作顺序执行
