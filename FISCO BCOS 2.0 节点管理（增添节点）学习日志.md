# FISCO BCOS 区块链网络 - 增加节点学习日志

## 一、 实验概述

**实验目标**：在现有的3节点FISCO BCOS PBFT共识网络中，需要成功添加第4个节点

**实验环境**：
- 阿里云ECS服务器 (4核8G)
- Ubuntu 22.04 操作系统  
- FISCO BCOS v2.11.0
- PBFT共识机制

**初始状态**：3节点共识网络正常运行
**目标状态**：4节点共识网络

---

## 二、 操作步骤与代码解析

### 第一步：生成新节点配置文件

```bash
# 生成第4个节点node3的配置文件
cd /root/fisco
./build_chain.sh -l "127.0.0.1:4" -p 30303,20203,8548 -o new_node_temp
```

**代码解释**：
- `-l "127.0.0.1:4"`：在本地地址生成4个节点
- `-p 30303,20203,8548`：指定P2P端口、Channel端口、JSON-RPC端口
- `-o new_node_temp`：输出到临时目录

**执行结果**：
```
[INFO] Start Port    : 30303 20203 8548
[INFO] Server IP     : 127.0.0.1:4  
[INFO] Output Dir    : /root/fisco/new_node_temp
```


---

### 第二步：复制新节点到网络目录

```bash
# 将生成的node3移动到节点目录
cp -r new_node_temp/127.0.0.1/node3 nodes/127.0.0.1/
```

**目录结构验证**：
```bash
ls -la nodes/127.0.0.1/
# 输出：node0  node1  node2  node3  ✅ 四个节点目录都存在
```

---

### 第三步：获取新节点身份标识

```bash
# 查看node3的节点ID（唯一身份标识）
cat nodes/127.0.0.1/node3/conf/node.nodeid
```

**输出结果**：
```
5640417050d2e60748348dc3301e5a8d6ee1a6c629de9362dd0bbb2123e6ae7e4109957199a9c89c67f7ec96a2b4b40e7e5361032fe3a3612296d751eb530d95
```

**代码解释**：
- 节点ID是256位的十六进制字符串
- 用于在网络中唯一标识每个节点
- 在P2P通信和共识投票中使用

---

### 第四步：配置现有节点识别新节点

#### 4.1 编辑node0配置
```bash
vim nodes/127.0.0.1/node0/config.ini
```

在 `[p2p]` 部分添加：
```ini
node.3=127.0.0.1:30303
```

#### 4.2 编辑node1配置
```bash
vim nodes/127.0.0.1/node1/config.ini
```
同样添加：
```ini
node.3=127.0.0.1:30303
```

#### 4.3 编辑node2配置  
```bash
vim nodes/127.0.0.1/node2/config.ini
```
同样添加：
```ini
node.3=127.0.0.1:30303
```

**配置原理**：
- PBFT共识需要所有节点相互知晓
- 每个节点的 `config.ini` 必须包含网络中所有其他节点的连接信息
- `node.3` 表示第4个节点的连接配置

---

### 第五步：配置新节点连接现有网络

```bash
vim nodes/127.0.0.1/node3/config.ini
```

修改 `[p2p]` 部分：
```ini
[p2p]
    listen_ip=0.0.0.0
    listen_port=30303
    ; nodes to connect
    node.0=127.0.0.1:30300
    node.1=127.0.0.1:30301  
    node.2=127.0.0.1:30302
```

修改 `[rpc]` 部分：
```ini
[rpc]
    channel_listen_ip=0.0.0.0
    channel_listen_port=20203
    jsonrpc_listen_ip=127.0.0.1
    jsonrpc_listen_port=8548
```

---

### 第六步：解决证书一致性问题

#### 问题发现
node3启动后出现SSL连接错误：
```
"tlsv1 alert unknown ca (SSL routines, ssl3 read bytes)"
```

#### 解决方案：统一CA证书
```bash
# 复制node0的证书给node3使用
cp nodes/127.0.0.1/node0/conf/ca.crt nodes/127.0.0.1/node3/conf/
cp nodes/127.0.0.1/node0/conf/node.crt nodes/127.0.0.1/node3/conf/
cp nodes/127.0.0.1/node0/conf/node.key nodes/127.0.0.1/node3/conf/
```

**证书验证**：
```bash
md5sum nodes/127.0.0.1/node0/conf/ca.crt
md5sum nodes/127.0.0.1/node3/conf/ca.crt
# 输出一致：b0caf22131... 
```

---

### 第七步：启动新节点

```bash
# 启动node3
./nodes/127.0.0.1/node3/start.sh
```

**启动验证**：
```bash
ps aux | grep fisco-bcos | grep node3
```
![[Pasted image 20251108054037.png]]
---

### 第八步：网络状态验证

#### 8.1 使用控制台检查
```bash
cd /root/fisco/nodes/127.0.0.1/console
./start.sh
```
![[Pasted image 20251108054139.png]]
#### 8.2 检查节点连接状态
```javascript
getGroupPeers
```

**输出结果**：
```json
{
    "051e4a13...",  // node0
    "4c70a2c2...",  // node1  
    "4d12374a...",  // node2
    "56404170..."   // node3 ✅ 成功连接
}
```

#### 8.3 添加为观察者节点
```javascript
addObserver 5640417050d2e60748348dc3301e5a8d6ee1a6c629de9362dd0bbb2123e6ae7e4109957199a9c89c67f7ec96a2b4b40e7e5361032fe3a3612296d751eb530d95
```

**验证观察者节点**：
```javascript
getObserverList
```
**输出**：包含node3的节点ID ✅
![[Pasted image 20251108054257.png]]
---

## 九、遇到的问题与解决方案

### 问题1：SSL证书不匹配
**现象**：节点间SSL握手失败
**原因**：node3使用新CA，其他节点使用旧CA
**解决**：统一使用相同的CA证书文件

### 问题2：配置格式错误  
**现象**：节点无法启动或连接
**原因**：端口配置错误、连接信息缺失
**解决**：仔细检查并修正所有节点的config.ini文件

### 问题3：共识节点转换失败
**现象**：观察者节点无法转为共识节点
**原因**：网络共识不稳定、证书验证问题
**状态**：仍在分析解决中

---

## 十、实验结果

###  1.成功完成
1. 新节点node3成功生成和配置
2. node3成功连接到现有网络
3. node3作为观察者节点正常运行
4. 所有节点间P2P通信正常

###  2.进行中
1. 观察者节点转为共识节点
2. 4节点PBFT共识稳定性
附：为什么在4节点FISCO BCOS网络中这个实验中，node3成功作为观察者节点加入，但无法转换为共识节点。
可能是PBFT共识机制限制
（**投票机制要求**：
4节点PBFT网络 → 需要 ≥3个节点同意（75%）
提议：添加node3为共识节点
需要：node0 ✅ + node1 ✅ + node2 ✅ 
实际：可能只有2个节点同意 → 投票失败）
还可能是证书信任体系问题
（**SSL/TLS握手流程**：
node3 → 其他节点: ClientHello (使用证书B)
其他节点 → node3: 证书验证失败 → 拒绝深度通信）
### 3. 网络状态
- **节点数量**：4个 (3共识 + 1观察者)
- **网络连接**：正常
- **数据同步**：正常
- **共识机制**：PBFT (3/4节点参与)

---

## 十一、学习总结

### 关键技术点
1. **节点身份管理**：每个节点有唯一的nodeID用于身份识别
2. **P2P网络配置**：所有节点必须相互配置连接信息
3. **证书安全管理**：统一的CA证书确保节点间安全通信
4. **共识机制理解**：PBFT需要≥75%节点同意才能达成共识

### 实践经验
- 节点添加需要保证配置的一致性
- 证书管理是节点间通信的关键
- 网络状态验证是必不可少的步骤
- 问题诊断需要系统性的排查方法
