# EXERCISES —— cdk diff 练习（从易到难）

目的:练「**改一处 → 预判 `cdk diff` 是 update 还是 replace**」的手感。
新架构(FastAPI + 读写分离 + ALB 锁定)下,练习已更新。

## 通用流程

```powershell
$env:AWS_REGION="ap-northeast-1"
# 1) 改代码  2) 看差异:
npx cdk diff
# 3) 对照下面「预期」核对预判  4) 可选:npx cdk deploy  5) 改回去保持干净
```

### 读 diff 符号

| 符号 | 含义 |
|------|------|
| `[~]` | 修改(可能 update,也可能触发 replace,看有无 replacement 警告) |
| `[+]` / `[-]` | 新增 / 删除资源 |
| `requires replacement` / `may be replaced` | **replace**:删旧建新 |

> 法则:**数量/容量/可调参数变 → update;物理标识/创建期属性变 → replace。**

---

## 练习 1（易）：改期望任务数 desiredCount

```diff
-      desiredCount: 1,
+      desiredCount: 2,
```
**预期**:`[~] AWS::ECS::Service`,`DesiredCount: 1 → 2`。**update**(原地)。
ECS 多拉一个任务并注册进 ALB 目标组,服务不重建。

---

## 练习 2（易→中）：改 Fargate CPU / 内存

```diff
-      cpu: 256,
-      memoryLimitMiB: 512,
+      cpu: 512,
+      memoryLimitMiB: 1024,
```
**预期**:`[~] AWS::ECS::TaskDefinition`。任务定义**不可变**,会生成**新修订版**,服务**滚动更新、不中断**。
> CPU/内存必须是 Fargate 合法组合(256/512、512/1024…),乱配 `synth` 会报错。

---

## 练习 3（中）：改 Aurora 最大容量

```diff
-      serverlessV2MaxCapacity: 1,
+      serverlessV2MaxCapacity: 2,
```
**预期**:`[~] AWS::RDS::DBCluster`,`ServerlessV2ScalingConfiguration.MaxCapacity: 1 → 2`。**update**(原地)。
在线调整伸缩上限,不重建集群、不丢数据。
> 对比:改 `engine` 大版本或 `defaultDatabaseName` 会是 **replace**(删库重建)——生产红线。

---

## 练习 4（中）：加 / 减一个 reader（贴合读写分离）

在 `readers` 数组里再加一个,或删掉现有的:
```diff
       readers: [
         rds.ClusterInstance.serverlessV2('reader1', { scaleWithWriter: true }),
+        rds.ClusterInstance.serverlessV2('reader2', { scaleWithWriter: true }),
       ],
```
**预期**:`[+] AWS::RDS::DBInstance`(新增 reader2),**不影响** writer 和 reader1。
- 加 reader = **新增实例**(不是 replace),Reader Endpoint 会自动把它纳入负载均衡。
- 删 reader = `[-]`,该实例被销毁。
- ⚠️ 每个 reader 都是一份 ACU 计费,练完记得减回去。
> 思考:为什么加 reader 是 `[+]` 而不是 replace?因为它是**往集群里加一个独立实例**,
> 现有实例的物理标识没变。这正是「加东西 vs 改身份」的区别。

---

## 练习 5（中→难）：改应用代码 → 镜像重建 → 新任务定义

改 [app/main.py](app/main.py),比如把页面标题或播种数据改一下:
```diff
-  <h1>🛒 商品一覧（Aurora 读写分离演示）</h1>
+  <h1>🛒 我的商品库（改了文案）</h1>
```
**预期**:`[~] AWS::ECS::TaskDefinition`,容器 `Image` 的 asset hash / tag **变化**。
**TaskDef 出新修订版,服务滚动更新。**

**这练习的价值(最该体会的)**:你只改了**应用代码**,没动任何 CDK 资源定义,但
`fromAsset` 会**重新算源码哈希 → 重新 `docker build` → 推新镜像 → 任务定义引用新 tag**。
也就是说 **「改 app 代码」本身就是一次基础设施变更**,deploy 时会重建镜像并滚动发布。
> 这是 IaC + 容器的典型链路:代码改动通过镜像哈希传导成 CloudFormation 变更。

**进阶思考(只 `cdk diff` 预判,别真部署)——什么会升级成 replace?**
1. 改容器端口 `containerPort: 8000 → 9000` → 目标组/服务层面较大改动。
2. 改 `defaultDatabaseName: 'appdb'` → DBCluster **replace**(删库重建)。
3. 改 VPC `maxAzs: 2 → 3` → 子网/路由表整体重排,大量新增。

> **面接**:「TaskDefinitionやイメージの変更はローリングアップデートで無停止ですが、DB名やエンジンのメジャーバージョン、サブネット構成のような作成時属性を変えると置き換え(replace)になり、データ損失や全断のリスクがあります。だからdeploy前に必ずcdk diffでrequires replacementを確認します。」

---

## 练完记得还原

```powershell
# 手动改回,确保 npx cdk diff 显示 “There were no differences”
```
