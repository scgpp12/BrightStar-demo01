# cdk-practice — CloudFront + ALB + ECS Fargate + Aurora Serverless v2(读写分离)

面试向 CDK（TypeScript）练习工程。容器 Web 架构,单 Stack 可部署、可销毁。
容器里跑一个 **FastAPI 商品查询小应用**,用来直观检验 Aurora 的**读写分离**。

```
用户 ─HTTPS─> CloudFront ─HTTP─> ALB ──> ECS Fargate(FastAPI:8000) ──┬─写→ Aurora writer 端点
            (viewer强制HTTPS)  (只收CloudFront)                       └─读→ Aurora reader 端点
                                                          Aurora Serverless v2 (writer + reader1, min=0)
```

- **入口**:只有 CloudFront。ALB 安全组锁成「只接受 CloudFront 回源」,公网**无法直连 ALB**。
- **应用**:`GET /` 列出商品(读 → reader 端点)、`POST /products` 新增商品(写 → writer 端点)。
- **读写分离**:页面会显示本次读落到了哪个节点(`pg_is_in_recovery`),肉眼可见读写分到不同实例。

---

## ⚠️ 成本警告（先读这个）

| 资源 | 计费方式 | 大致单价 |
|------|----------|----------|
| **NAT Gateway** | 按小时 + 流量 | ~$0.045 / 小时 |
| **ALB** | 按小时 + LCU | ~$0.025 / 小时 |
| CloudFront | 按请求/流量 | 练习量基本可忽略 |
| Aurora Serverless v2 ×2(writer+reader) | 按 ACU·小时 | min=0,闲时**两个都自动暂停**,几乎不花钱 |
| ECR 镜像存储 | 按 GB·月 | 几分钱,可忽略 |

> 固定成本约 **$0.1 / 小时**(NAT+ALB)。挂一个月 **$150+**。
> **每次练完必须 `cdk destroy`。** 加了 reader 后是**两个** Aurora 实例,但 min=0 闲时都会缩到 0。

---

## 前提环境

- **Node.js 20+**(本工程用 24 验证过)。
- **Docker Desktop 必须装好并正在运行** ⭐ —— 因为商品应用是自建镜像,`cdk deploy` 会在本机
  `docker build` 再推到私有 ECR。部署前用 `docker info` 确认 Docker 在跑。
- **AWS 凭证**:`aws sts get-caller-identity` 能返回账号即 OK。
- **区域**:本练习部署在**东京 `ap-northeast-1`**。新开终端先设:
  ```powershell
  $env:AWS_REGION="ap-northeast-1"
  ```
- **CDK bootstrap**(每个 账号×区域 一次):`npx cdk bootstrap`。

---

## 完整命令流程

```powershell
$env:AWS_REGION="ap-northeast-1"

# 1) 安装依赖
npm install

# 2) 合成模板(本地检查;镜像在 deploy 阶段才 build,synth 不需要 Docker)
npx cdk synth

# 3) 看将要发生的变更
npx cdk diff

# 4) 部署(首次会 docker build + 推 ECR + 建 reader,见下方耗时说明)
npx cdk deploy --require-approval never
```

### 部署要等多久？别以为卡住了

- **首次构建镜像**:本机 `docker build`(拉 python 基础镜像 + 装依赖)+ 推 ECR,约 2~5 分钟。
- **Aurora**:writer + reader 两个实例,约 5~10 分钟。
- **CloudFront**:5~15 分钟(最慢)。
- 全程预期 **15~25 分钟**。终端进度数字在动就是正常。

### 部署后如何验证(读写分离)

部署成功后,终端 `Outputs:` 打印:

```
CdkPracticeStack.CloudFrontURL = https://xxxxxxxx.cloudfront.net
CdkPracticeStack.DbSecretName  = CdkPracticeStackAuroraSecre-xxxx
```

1. 浏览器打开 **CloudFrontURL** → 看到「🛒 商品一覧」页面,顶部徽章显示 **「只读副本 reader ✅」**。
   - 闲置后首次打开可能慢几秒(Aurora min=0 在唤醒),应用内置了重试,稍等即可。
2. 用页面表单**新增一个商品**(写 → writer)→ 自动跳回列表。
3. **刷新一两次**:刚加的商品出现。若第一次没看到、第二次才看到 → 证明读(reader)和写(writer)
   落在不同节点,中间有复制延迟。**这就是读写分离的肉眼证据。**

> 注意:**直连 ALB 会失败**(连接被安全组拒绝)。这是设计如此 —— 唯一入口是 CloudFront。

### 销毁（练完必做）

```powershell
$env:AWS_REGION="ap-northeast-1"; npx cdk destroy
```

---

## 如何确认删干净

`cdk destroy` 跑完后去控制台二次确认(重点是会花钱的):

1. **CloudFormation** → `CdkPracticeStack` 已消失(或 `DELETE_COMPLETE`)。
2. **VPC → NAT 网关**:无残留(**最贵,重点看**)。
3. **VPC → 弹性 IP**:NAT 的 EIP 已释放。
4. **EC2 → 负载均衡器**:ALB 已不存在。
5. **RDS**:两个 Aurora 实例 + 集群都已删除。
6. **VPC 列表**:练习 VPC 已删除。

> 上面都能用标签 `project=cdk-practice` / `deploy=aws-learn01` 快速筛。
> 💡 小例外:`fromAsset` 推到 **bootstrap 的 ECR 仓库**(`cdk-hnb659fds-container-assets-...`)的镜像,
> `cdk destroy` **不会删**。它们很小(几分钱),不影响。要清理可去 ECR 控制台手动删那个仓库里的旧镜像。

---

## 工程结构

```
.
├── app/                        # 商品查询小应用(自建镜像,推到 ECR)
│   ├── main.py                 # FastAPI:读走 reader、写走 writer
│   ├── requirements.txt
│   └── Dockerfile
├── bin/app.ts                  # CDK 入口 + 标签(project / deploy)
├── lib/cdk-practice-stack.ts   # 全部架构 + L1/L2/L3 注释 + escape hatch + ALB 锁定
├── WALKTHROUGH.md              # 部署后控制台巡检清单(含日语面试话术)
├── EXERCISES.md                # cdk diff 练习(update vs replace)
├── AWS-NOTES.md                # AWS 知识点笔记(本次实战沉淀,面试复习用)
├── cdk.json / package.json / tsconfig.json
```

下一步:部署后照 [WALKTHROUGH.md](WALKTHROUGH.md) 巡检,再用 [EXERCISES.md](EXERCISES.md) 练 `cdk diff`。
