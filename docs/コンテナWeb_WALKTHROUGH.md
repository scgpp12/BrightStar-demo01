# WALKTHROUGH —— 部署后控制台巡检清单

`cdk deploy` 成功后,照这份清单去 AWS 控制台(区域切到**东京 ap-northeast-1**)逐项确认。
顺序按构成图:**网络 → 数据库(读写分离)→ 镜像 → 容器 → 入口安全(锁 CloudFront)→ 密钥 → CDN**。
每个点附一句**日语面试话术**。

---

## 1. VPC：三种子网的路由表差异

**路径**:VPC 控制台 → 子网 / 路由表

| 子网 | 0.0.0.0/0 指向 | 放什么 |
|------|----------------|--------|
| `public` ×2 | **IGW** | ALB |
| `private-egress` ×2 | **NAT** | Fargate 任务 |
| `isolated` ×2 | **没有**(只有 local) | Aurora |

重点确认 isolated 子网路由表**只有 local、没有出口**。

> **面接**:「パブリックはIGW、プライベートはNAT、アイソレートは出口なし。DBは出口のないアイソレートに置いて攻撃面を最小化しています。」

---

## 2. NAT Gateway（花钱的东西）

**路径**:VPC → NAT 网关 / 弹性 IP。应只有 **1 个** NAT + 1 个 EIP。

> **面接**:「コスト削減のためNATは1つだけ。本番ではAZごとに置きますが、NATは起動しているだけで時間課金されるので学習では絞っています。」

---

## 3. Aurora：写实例 + 读副本(读写分离的核心)

**路径**:RDS 控制台 → 数据库 → 点开集群

确认点:
- **集群下有两个实例**:`...writer`(写) + `...reader1`(读)。角色列分别是 **Writer / Reader**。
- **配置**:Aurora PostgreSQL 16,容量类型 **Serverless v2**,容量范围 **0~1 ACU**(min=0 闲时暂停)。
- **连接与安全**里有两个端点:
  - **集群端点(Cluster Endpoint)** → 永远指向 writer(应用写用这个)
  - **读取器端点(Reader Endpoint)** → 在 reader 间负载均衡(应用读用这个)
- **子网组**:全是 isolated 子网。
- **维护**:扩展支持(Extended Support)关闭(escape hatch 设的 `EngineLifecycleSupport`)。

> **面接**:「Auroraは計算とストレージが分離していて、writerは書き込み専用、readerは読み取り用です。アプリはクラスターエンドポイントに書き込み、リーダーエンドポイントから読み取ることで読み書きを分離しています。Serverless v2でMin 0なので、アイドル時は両インスタンスとも自動停止します。」

---

## 4. ECR：自建镜像仓库(这次新增)

**路径**:ECR 控制台 → 仓库 → `cdk-hnb659fds-container-assets-...`

确认点:里面有一个刚 push 上去的镜像(tag 是一长串 hash)。这就是 `cdk deploy` 时
本机 `docker build ./app` 后自动推上来的 FastAPI 镜像,ECS 任务从这里拉取。

> **面接**:「アプリは自前のイメージで、cdk deploy時にローカルでdocker buildして、CDKが自動的にプライベートECRにpushします。Fargateはそのイメージを取得して起動します。」

---

## 5. ECS 任务定义：端点(明文)+ 口令(密钥)的混合注入

**路径**:ECS → 任务定义 → 最新修订 → 容器 `web` → 环境变量

确认点(两类注入并存):
- **环境变量(Environment,明文)**:`DB_WRITER_HOST` / `DB_READER_HOST` —— 是两个端点主机名。
  主机名不是机密,所以用明文环境变量。
- **机密(Secrets,ValueFrom)**:`DB_USER` / `DB_PASSWORD` / `DB_NAME` / `DB_PORT` ——
  值是 `密钥ARN:字段::`,口令不出现明文,启动时由执行角色从 Secrets Manager 取。

> **面接**:「エンドポイントのホスト名は機密ではないので環境変数で、パスワードなどの認証情報はSecrets ManagerのARN参照(valueFrom)で注入しています。平文とSecretを使い分けています。」

---

## 6. ALB 目标组：健康检查走 /healthz，端口 8000

**路径**:EC2 → 目标组 → 点开 → 目标 / 健康检查

确认点:
- 注册的 Fargate 任务 IP 状态 **healthy**(注册端口是容器的 **8000**)。
- 健康检查路径是 **`/healthz`**(不是 `/`)。故意用一个**不查库**的轻量端点 ——
  因为 Aurora min=0 暂停时查库会变慢,若健康检查依赖 DB 可能误判 unhealthy 反复重启。

> **面接**:「ヘルスチェックはDBに触れない軽量な/healthzにしています。Serverless v2はアイドル時に停止するので、ヘルスチェックがDBに依存すると誤ってunhealthy判定になりタスクが再起動を繰り返す恐れがあるためです。」

---

## 7. ⭐ ALB 安全组：只接受 CloudFront(这次新增,面试加分)

**路径**:EC2 → 安全组 → ALB 的安全组 → 入站规则

确认点:80 端口入站规则的**「来源」是一个托管前缀列表 `pl-xxxx`**(CloudFront origin-facing),
**不是 `0.0.0.0/0`**。所以公网即使知道 ALB 域名也连不上,流量只能经 CloudFront 进。

验证:直接 `curl http://<ALB域名>` 会**超时/被拒**;只有 CloudFront URL 能访问。

> **面接**:「ALBはパブリックですが、セキュリティグループのインバウンドをCloudFrontのマネージドプレフィックスリストに限定しているので、CloudFrontを経由しない直接アクセスはできません。オリジンへの直アクセスによるWAFやHTTPSの迂回を防いでいます。」

---

## 8. DB 安全组 5432：来源是 ECS 的安全组

**路径**:EC2 → 安全组 → Aurora 的安全组 → 入站

确认点:5432 入站的「来源」是 **Fargate 服务的安全组 ID**(不是 IP)。writer/reader 共用同一集群安全组。

> **面接**:「DBの5432はソースがFargateサービスのセキュリティグループIDで、そのサービスからのみ接続を許可する最小権限構成です。」

---

## 9. Secrets Manager：自动生成的密钥

**路径**:Secrets Manager → `CdkPracticeStackAuroraSecre-...`(Output 的 `DbSecretName`)→ 检索密钥值

确认点:JSON 里 `username/password/host/port/dbname` 自动生成,`password` 是 30 位随机串,你从没写过。

> **面接**:「パスワードはコードに書かず、CDKが自動生成してSecrets Managerに保存します。ローテーションも設定可能で、認証情報をリポジトリに置くリスクがありません。」

---

## 10. CloudFront：源指向 ALB

**路径**:CloudFront → 分配 → 源 / 行为

确认点:
- **源**:ALB 的 DNS,协议 **仅 HTTP(HTTP only)**。
- **行为**:缓存策略 **CachingDisabled**(演示读写,不缓存);查看器协议 **重定向到 HTTPS**。

> **面接**:「CloudFrontのオリジンはALBで、ALBは80のみなのでHTTP onlyです。今回は読み書きの動作確認のためキャッシュは無効化、ビューワーはHTTPSにリダイレクトしています。」

---

## 巡检完了の一言（面接の締め）

> 「CloudFront → ALB → ECS Fargate → Aurora Serverless v2 の典型的なコンテナWeb構成です。Auroraは読み書き分離、ALBはCloudFront経由のみに制限、認証情報はSecrets Manager、最小権限のセキュリティグループ、そしてServerless v2 Min 0でコスト最適化、という点を意識して設計しました。」
