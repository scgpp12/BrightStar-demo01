# AWS 知识点笔记（本次实战沉淀）

这份是围绕 **CloudFront + ALB + ECS Fargate + Aurora Serverless v2** 这套架构，在搭建/调试/验证过程中聊到的 AWS 知识点汇总，按主题归类，附**面接で使える日语**。

---

## 1. CDK 三层 construct + escape hatch

- **L1（Cfn*）**：和 CloudFormation 资源 1:1，参数全暴露但全要自己填。
- **L2**（如 `Vpc`/`DatabaseCluster`）：带默认值和便捷方法的封装，日常主力。
- **L3**（如 `ApplicationLoadBalancedFargateService`）：把一组 L2 编排成一个模式，**一行展开几十个资源**（本例那一行 = 14 个资源）。
- **Escape Hatch**：L2 没暴露的属性，下沉到 L1 改。手法：
  ```ts
  const cfn = db.node.defaultChild as rds.CfnDBCluster;
  cfn.addPropertyOverride('EngineLifecycleSupport', 'open-source-rds-extended-support-disabled');
  ```
  > 「L2が公開していないプロパティは、defaultChildでL1のCfnリソースを取り出してaddPropertyOverrideで設定します。」

## 2. CDK deploy 的 6 步流水线

1. 执行 app（跑你的 TS，搭 Construct 树）→ 2. **synth**（树 → CFN 模板 + 资产）→ 3. 算 diff/审批 → 4. 发布资产到 bootstrap 的 S3 → 5. CreateChangeSet + ExecuteChangeSet（**CloudFormation 真正建资源**）→ 6. 回传 Outputs。
- **CDK 本身不创建资源**，只生成模板 + 协调部署；建资源的永远是 CloudFormation。
- **bootstrap（CDKToolkit 栈）是前提**：提供 S3 资产桶 + 几个 IAM 角色（FilePublishingRole / CloudFormationExecutionRole 等）。

## 3. cdk diff：update vs replace

- **update（原地）**：数量/容量/可调参数变 —— `desiredCount`、Aurora `MaxCapacity`、健康检查、加 reader（`[+]`）。
- **replace（删旧建新）**：物理标识/创建期属性变 —— `defaultDatabaseName`、引擎大版本、子网构成。**生产红线，部署前必看 `requires replacement`。**
- **TaskDefinition 不可变**：改任何字段（含镜像）都生成**新修订版（revision+1）**，服务滚动更新、不中断。
  > 「DB名やエンジンのメジャーバージョンのような作成時属性を変えると置き換えになり、データ損失のリスクがあります。」

---

## 4. CloudFront

- **作用**：边缘缓存加速 / **HTTPS 终结（免费证书）** / 隐藏源站 / Shield Standard 防 DDoS / 就近接入。
- **计费**：按出站流量 + 请求数，**缓存本身不单收费**；有永久免费层（1TB/月 + 1000万请求）。开缓存通常**更省钱**（命中不回源）。
- **默认行为只允许 GET/HEAD** ⚠️：POST/PUT/DELETE 会被挡成 **403**。可写 API 必须 `allowedMethods: ALLOW_ALL`。**（本次实战踩到的坑）**
- `CACHING_DISABLED`：每次回源，调试用；`X-Cache: Miss from cloudfront` 可确认没缓存。
- **排查「部署了新版本浏览器还显示旧的」**：先硬刷新/无痕排除**浏览器缓存**，再看 CloudFront `X-Cache`，最后才怀疑源站。
  > 「CloudFrontはデフォルトでGET/HEADしか許可しないので、書き込みAPIを通すにはallowedMethodsを広げる必要があります。」

## 5. CloudFront → ALB 的安全

- **「走 AWS 内网骨干」≠ 安全**：仍是**明文 HTTP**、骨干网多租户共享；合规（PCI-DSS 等）不认「在 AWS 内网」作为不加密的理由。
- 短板：① 回源明文 ② **ALB 公网可被直连绕过 CloudFront**。
- **免费锁源方案（本例用的）**：ALB 安全组只放行 **CloudFront 托管前缀列表** `com.amazonaws.global.cloudfront.origin-facing`，公网直连被拒。CDK 用 `AwsCustomResource` 查前缀列表 ID + `openListener:false`。
- 真·全程 HTTPS 方案：ALB 上 ACM 证书 + 443 监听 + 回源 HTTPS（需自定义域名）。
  > 「ALBはパブリックですが、セキュリティグループをCloudFrontのマネージドプレフィックスリストに限定して、直接アクセスを防いでいます。」

## 6. ACM / Route 53

- **ACM 公有证书**：配合 CloudFront/ALB 使用**免费**，自动续期免费；但**必须签给你拥有的域名**（`*.elb.amazonaws.com` 签不了）。**证书免费 ≠ 域名免费。**
- **ACM Private CA**：企业内部 PKI（mTLS、内网域名、IoT），**约 $400/月**，练习用不到。
- **Route 53**：只有要**自定义域名**时才需要（Alias 记录指向 CloudFront/ALB）；AWS 默认 DNS 名不需要。托管区 $0.50/月，指向 AWS 资源的 Alias 查询免费。

---

## 7. ALB（七层负载均衡）

- `internet-facing` + **跨 2 AZ**（自带高可用，单 AZ 挂自动切）。
- **Listener**：监听端口/协议 + 转发规则（可按路径/域名/头分发）。
- **TargetGroup**：`TargetType: ip`（Fargate `awsvpc` 每任务独立 IP，所以是 ip 不是 instance）；**健康检查**只往 healthy 目标转流量。
- **安全组分层**：ALB 对外开放，后端只认 ALB（安全组引用）。
- **ECS 自动维护目标组**：任务起/停自动注册/注销 IP —— 负载均衡靠这层胶水。
  > 「FargeteはawsvpcモードでタスクごとにIPを持つので、ターゲットタイプはipです。ECSがタスクの増減に合わせて登録・解除を自動で行います。」

## 8. ECR vs ECS

- **ECR**：镜像仓库（AWS 版 Docker Hub）。**公共 ECR**（`public.ecr.aws`，免授权）vs **私有 ECR**（IAM 授权、扫描、加密）。
- **ECS**：容器编排。Cluster → Service（保活/滚动更新）→ Task（任务定义的运行实例）。
- **启动类型**：**Fargate**（AWS 管底层，serverless 容器）vs EC2（自己管节点）。
- **公共镜像免 ECR 权限；私有 ECR 镜像**需执行角色加 `ecr:GetAuthorizationToken`/`BatchGetImage` 等（CDK 自动 grant）。**（本次 diff 里亲眼看到）**

## 9. Task Definition（任务定义）

- **不可变 + 版本化**（revision）。蓝图含：镜像、CPU/内存、端口、环境变量、secrets、两个角色、网络模式。
- **两类注入**：
  - `environment`（明文）：放**非机密**，如 DB 端点主机名。
  - `secrets`（`valueFrom`）：放**机密**，值是 `密钥ARN:字段::`，启动时执行角色从 Secrets Manager 取。
- **两个角色分工**：
  - **ExecutionRole**（启动期）：拉镜像、取密钥、写日志（ECS 自己用）。
  - **TaskRole**（运行期）：容器内 app 调 AWS API 用（本例为空，nginx/FastAPI 不调 AWS）。
- `networkMode: awsvpc` → 每任务独立 ENI/IP（与 ALB target=ip 呼应）。

## 10. Secrets Manager

- `fromGeneratedSecret`：**自动生成强密码**存入，代码里没有明文。
- **按字段注入 `, 'password'` 只影响「注入哪个值」，不缩小 IAM 权限**：Secrets Manager **不支持字段级 IAM**，`GetSecretValue` 永远是整条密钥。要字段级隔离只能拆成多条密钥。**（本次做过对照实验验证）**
- **验证「默认 grant 对不对」**：读 synth 模板里的 `AWS::IAM::Policy` —— 三件套：Action 最小、Resource 限定到具体 ARN（非 `*`）、Effect=Allow。

## 11. Secrets Manager vs Parameter Store

| | Secrets Manager | Parameter Store |
|---|---|---|
| 定位 | 机密专用 | 通用配置（也能存机密） |
| 自动轮换/生成 | ✅ | ❌ |
| 费用 | $0.40/密钥/月 | 标准版**免费** |
| RDS 集成 | ✅ 原生 | ❌ |

实践：**要轮换的凭证走 Secrets Manager，普通配置走 Parameter Store。** ECS 两者都能注入。

---

## 12. Aurora Serverless v2

- **不是 RDS 跑 PG**，是 AWS 重写存储引擎、兼容 PG 协议。**计算与存储分离**；数据自动跨 3 AZ 存 6 副本。
- **Serverless v2**：按 **ACU** 自动伸缩；**`min=0` 闲时自动暂停**（省钱，代价是冷启动几秒）。
- **至少一个 writer**：存储层不执行 SQL，必须有计算实例；单写架构，writer 唯一可写。
- **读写分离**：
  - **Cluster Endpoint（写端点）** → 永远指向 writer。
  - **Reader Endpoint（读端点）** → 多 reader 间负载均衡。
  - **DB 不自动分流**，是**应用**决定读连读端点、写连写端点。
  - 加 reader 还带来**故障自动切换**（writer 挂了 reader 自动升主）。
- **加 reader = `[+]` 新增实例**（不是 replace）；每个 reader 一份 ACU 计费。
- 练习设 `removalPolicy: DESTROY` + `deletionProtection: false`（生产相反）。

## 13. 验证数据是否真在库里（isolated 子网怎么查）

证据由弱到强：① 访问日志（只证请求成功）② 页面能列出（应用读到了）③ **直接查库**（最硬）。
- Aurora 在 isolated 子网无直连，**直接查库三法**：
  1. **RDS Data API**（本例用的，最轻量，走 AWS HTTPS 无需网络）：`enableDataApi: true`，然后 `aws rds-data execute-statement --resource-arn <集群> --secret-arn <密钥> --database appdb --sql "..."`。
  2. 跳板机/堡垒机 + psql。
  3. ECS Exec 进容器查（需 `enableExecuteCommand`）。
  > 「アイソレートサブネットのAuroraを直接確認するには、RDS Data APIが一番手軽です。ネットワークに入らずHTTPS経由でSQLを実行できます。」

---

## 14. 网络 / VPC

- 三类子网路由差异：**public→IGW、private-egress→NAT、isolated→只有 local（无出口）**。
- DB 放 isolated（无出口最安全）；Fargate 放 private-egress（能出网拉镜像）；ALB 放 public。
- **安全组按引用放行**（非 IP）：`db.connections.allowDefaultPortFrom(svc.service)` = 「DB 5432 入站，来源 = 服务安全组」，最小权限。

## 15. CloudWatch Logs

- 容器 stdout/stderr 经 `awslogs` 驱动写入 CloudWatch 日志组。
- 看法：控制台 CloudWatch → 日志组；或 `aws logs tail <组名> --follow --since 5m --region ...`。
- **访问日志的源 IP 是 ALB 内网 IP，不是真实访客**；真实访客 IP 要看 `X-Forwarded-For`。
- 健康检查日志来自**两个 ALB 节点 IP**（2 AZ）。

## 16. 成本意识

- **持续烧钱**：NAT Gateway（~$0.045/h）+ ALB（~$0.025/h）≈ **$0.1/h**，与用不用无关。
- **几乎不烧**：Aurora Serverless v2 `min=0`（闲时暂停）、CloudFront（免费层内）、ACM 公有证书（免费）。
- **练完必 `cdk destroy`**；bootstrap 的 CDKToolkit 栈保留（便宜、可复用）。
- 残留注意：`fromAsset` 推到 bootstrap ECR 仓库的镜像 `cdk destroy` 不删（很小）。

---

## 附：本次实战踩的两个真实坑

1. **CloudFront 默认 GET/HEAD-only → POST 403**：可写 API 必须 `allowedMethods: ALLOW_ALL`。
2. **浏览器缓存导致看到旧 nginx 页**：硬刷新/无痕解决；curl 一直对是因为它无本地缓存。
（外加 Windows 控制台 GBK 印不出日文 → `PYTHONUTF8=1` / `chcp 65001`；bash 与 PowerShell 变量语法不同别混用。）
