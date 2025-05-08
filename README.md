# 自动添加控制台登录 IP 到安全组白名单 - 架构图

以下是该解决方案的架构图，展示了从用户登录到安全组更新的完整流程：

```
┌─────────────────┐                ┌───────────────┐                ┌───────────────────┐
│                 │                │               │                │                   │
│  IAM Identity   │───Console────▶│  AWS Console  │────Login─────▶ │  CloudTrail       │
│  Center User    │    Login       │               │     Event      │(ConsoleLoginTrail)│
└─────────────────┘                └───────────────┘                └──────────┬────────┘
                                                                               │
                                                                               │
                                                                               │ Logs Events
                                                                               ▼
┌─────────────────┐                ┌───────────────┐                ┌───────────────────┐
│                 │                │               │                │                   │
│  EC2 Security   │◀───Update─────│  Lambda       │◀───Trigger─────│  EventBridge      │
│  Group          │   Security     │  Function     │                │      Rule         │
│                 │   Group Rules  │               │                │                   │
└─────────────────┘                └───────────────┘                └──────────┬────────┘
                                                                               │
                                                                               │
                                                                               │
                                                                               │
┌─────────────────┐                ┌───────────────┐                ┌──────────▼────────┐
│                 │                │               │                │                   │
│  S3 Bucket      │◀───Archive────│  CloudWatch   │◀───Stream──────│  CloudTrail Logs  │
│  (Log Storage)  │    Logs        │  Logs Group   │     Logs       │                   │
│                 │                │  (7-day       │                │                   │
└─────────────────┘                │   retention)  │                └───────────────────┘
                                   └───────────────┘
```

## 架构流程说明

1. **用户登录触发事件**
   - IAM Identity Center 用户通过 AWS 控制台登录
   - AWS 控制台生成登录事件

2. **CloudTrail 捕获登录事件**
   - 专用的 CloudTrail (ConsoleLoginTrail) 捕获控制台登录事件
   - 登录事件被记录并保存

3. **事件处理流程**
   - CloudTrail 将日志发送到 CloudWatch Logs 组（保留 7 天）
   - CloudTrail 日志也被归档到 S3 存储桶（保留 30 天）
   - EventBridge 规则监控特定用户的成功登录事件

4. **自动化响应**
   - 当检测到匹配的登录事件时，EventBridge 规则触发 Lambda 函数
   - Lambda 函数从事件中提取源 IP 地址
   - Lambda 函数更新指定的 EC2 安全组，添加该 IP 地址的入站规则
   - 如果安全组中已有超过配置的最大 IP 数量，Lambda 会移除最旧的 IP

## 关键组件

- **CloudTrail**: 专门配置用于捕获控制台登录事件
- **CloudWatch Logs**: 存储 CloudTrail 日志，保留期为 7 天以降低成本
- **S3 存储桶**: 长期归档 CloudTrail 日志，保留期为 30 天
- **EventBridge 规则**: 过滤特定用户的成功登录事件
- **Lambda 函数**: 处理登录事件并更新安全组规则
- **EC2 安全组**: 被动态更新以允许来自用户当前 IP 的 SSH 和 RDP 访问

这个架构实现了一个完全自动化的流程，当指定的 IAM Identity Center 用户登录 AWS 控制台时，其 IP 地址会被自动添加到安全组的白名单中，从而提高了安全性和便利性。

## 先决条件

1. **AWS 账户访问权限**
   - 需要具备创建和管理 CloudFormation 堆栈的权限
   - 需要具备创建 IAM 角色、CloudTrail、Lambda 函数等资源的权限

2. **已存在的安全组**
   - 必须已经创建好要动态更新的 EC2 安全组
   - 记录安全组 ID 用于部署时配置

3. **IAM Identity Center 配置**
   - 确定需要监控的 IAM Identity Center 用户邮箱/用户名
   - 确保该用户有权限通过控制台登录

4. **AWS CLI（可选）**
   - 如果使用命令行部署，需要安装和配置 AWS CLI

## 部署步骤

### 通过 AWS 控制台部署

1. **下载 CloudFormation 模板**
   - 下载 `auto-ip-whitelist.yaml` 文件到本地

2. **登录 AWS 控制台**
   - 导航到 CloudFormation 服务
   - 确保已准备好在两个中国区域部署

3. **创建主区域堆栈 (如 cn-northwest-1)**
   - 点击 "创建堆栈" > "使用新资源"
   - 选择 "上传模板文件"，然后上传 `auto-ip-whitelist.yaml`
   - 点击 "下一步"

4. **配置堆栈参数**
   - 输入堆栈名称（例如 `AutoAddIPWhitelist`）
   - 配置以下参数：
     - `UserName`: 要监控的 IAM Identity Center 用户名（格式如 `user@example.com`）
     - `SecurityGroupId`: 要更新的安全组 ID
     - `CloudTrailName`: CloudTrail 名称（可使用默认值）
     - `SecondaryRegion`: 次要区域（如 `cn-north-1`）
     - `PrimaryRegion`: 主区域（如 `cn-northwest-1`，即安全组所在区域）
     - `MaxIPs`: 安全组白名单中保留的最大 IP 数量（默认为 3）
   - 点击 "下一步"

5. **配置堆栈选项**
   - 保留默认设置或根据需要调整
   - 点击 "下一步"

6. **审核堆栈**
   - 检查所有配置是否正确
   - 勾选 "我确认 CloudFormation 可能会创建 IAM 资源" 复选框
   - 点击 "创建堆栈"

7. **在次要区域部署堆栈（如 cn-north-1）**
   - 登录次要区域的控制台
   - 重复步骤 3-6，使用完全相同的参数值
   - 特别注意参数需保持一致：
     - `SecurityGroupId`: 与主区域相同（安全组所在区域的 ID）
     - `SecondaryRegion`: 与主区域相同（如 `cn-north-1`）
     - `PrimaryRegion`: 与主区域相同（如 `cn-northwest-1`）

### 通过 AWS CLI 部署

1. **准备部署参数**
   - 创建参数文件 `parameters.json`：

     ```json
     [
       {
         "ParameterKey": "UserName",
         "ParameterValue": "user@example.com"
       },
       {
         "ParameterKey": "SecurityGroupId",
         "ParameterValue": "sg-xxxxxxxxxxxxxxxxx"
       },
       {
         "ParameterKey": "CloudTrailName",
         "ParameterValue": "ConsoleLoginTrail"
       },
       {
         "ParameterKey": "SecondaryRegion",
         "ParameterValue": "cn-north-1"
       },
       {
         "ParameterKey": "PrimaryRegion",
         "ParameterValue": "cn-northwest-1"
       },
       {
         "ParameterKey": "MaxIPs",
         "ParameterValue": "3"
       }
     ]
     ```

2. **在主区域(如 cn-northwest-1)部署堆栈**
   ```bash
   aws cloudformation create-stack \
     --stack-name AutoAddIPWhitelist \
     --template-body file://auto-ip-whitelist.yaml \
     --parameters file://parameters.json \
     --capabilities CAPABILITY_IAM \
     --region cn-northwest-1
   ```

3. **在次要区域(如 cn-north-1)部署堆栈**
   ```bash
   # 使用完全相同的参数文件，无需修改
   aws cloudformation create-stack \
     --stack-name AutoAddIPWhitelist \
     --template-body file://auto-ip-whitelist.yaml \
     --parameters file://parameters.json \
     --capabilities CAPABILITY_IAM \
     --region cn-north-1
   ```

## 部署关键点

1. **参数保持一致**
   - 在两个区域使用完全相同的参数值
   - `SecurityGroupId` 应始终指向主区域的安全组
   - `SecondaryRegion` 在两个部署中都应设为次要区域（如 `cn-north-1`）
   - `PrimaryRegion` 在两个部署中都应设为主区域（如 `cn-northwest-1`）

2. **部署逻辑**
   - Lambda 函数通过 `PrimaryRegion` 参数确定目标安全组所在区域
   - 无论 Lambda 函数在哪个区域执行，它都会更新主区域的安全组
   - 无需复杂的跨区域配置，避免 AWS 中国区域的跨区域限制

## 验证部署

1. **检查资源创建**
   - 在 CloudFormation 控制台确认两个区域的堆栈都创建成功
   - 查看 "资源" 选项卡确认所有资源都已创建
   - 查看 "输出" 选项卡，确认 `DeploymentMode` 显示正确的部署模式

2. **测试功能**
   - 使用指定的 IAM Identity Center 用户登录 AWS 控制台（尝试从两个不同区域的终端登录）
   - 等待约 1 分钟
   - 检查指定的安全组是否已添加您当前的 IP 地址
   - 确认新添加的入站规则允许 SSH (22) 和 RDP (3389) 访问

## 注意事项

1. **AWS 中国区域特殊限制**
   - 中国区域不支持跨区域的 EventBridge 规则目标
   - 此方案通过固定 Lambda 目标区域而非使用跨区域 EventBridge 规则来解决此问题

2. **安全考虑**
   - 该解决方案添加的安全组规则允许 SSH 和 RDP 从用户的 IP 地址访问
   - 请确保您的环境对这些协议有额外的保护措施（如强密码策略）

3. **成本优化**
   - CloudTrail 日志保留在 S3 中 30 天，在 CloudWatch Logs 中 7 天
   - 如需长期保留，可以修改模板中的保留策略

4. **最大 IP 限制**
   - 默认情况下，Lambda 函数最多保留 3 个 IP 地址
   - 现在可以通过 `MaxIPs` 参数在部署时指定最大保留的 IP 数量

5. **两个区域的事件捕获**
   - 由于在两个区域分别部署堆栈，无论用户在哪个区域登录，事件都将被捕获
   - 每个区域部署的 EventBridge 规则只能捕获自己区域的登录事件

6. **用户名匹配注意事项**
   - 对于 IAM Identity Center 用户，用户名通常是其电子邮件地址
   - 在检测用户登录时，系统会从 `principalId` 中提取用户名（格式通常为 `AROAXXXXX:user@example.com`）
   - 请确保在部署时提供的 `UserName` 参数与 IAM Identity Center 中配置的用户名完全匹配

7. **调试问题**
   - 查看 Lambda 函数的 CloudWatch 日志以排查问题
   - 查看 Lambda 环境变量，确保 `TARGET_REGION` 和 `USER_NAME` 设置正确

## 清理资源

如果您不再需要此解决方案，可以通过以下步骤删除所有资源：

1. **通过控制台删除**
   - 登录 AWS 控制台，导航到 CloudFormation 服务
   - 选择堆栈 `AutoAddIPWhitelist`
   - 点击 "删除"，然后确认删除
   - 在另一个区域重复相同的步骤

2. **通过 AWS CLI 删除**
   ```bash
   aws cloudformation delete-stack --stack-name AutoAddIPWhitelist --region cn-northwest-1
   aws cloudformation delete-stack --stack-name AutoAddIPWhitelist --region cn-north-1
   ```

**注意**: S3 存储桶使用 `DeletionPolicy: Retain` 策略，因此在删除堆栈后不会自动删除。如需删除 S3 存储桶，需要手动操作。
