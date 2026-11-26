# Vast.ai：API、CLI 与 Agent 租卡工作流

更新日期：2026-11-26。核验范围：官方在线文档、官方代码仓库说明，以及使用本地凭据进行的只读实例列表查询。没有创建、修改或销毁实例。

## 要解决的问题

本项目使用 Vast.ai 租用 GPU，希望 Agent 能操作算力资源，减少手动操作控制台。

Vast.ai 提供官方 REST API、CLI 和 Python SDK。官方还提供面向编码 Agent 的技能与插件。租卡自动化有可用的官方接口，无需依赖网页点击。

## 接入方式

| 方式 | 用途 | 本项目建议 |
| --- | --- | --- |
| `vastai` CLI | Agent 调用命令，查询报价、管理实例 | 先用它建立最小工作流；`--raw` 提供可解析 JSON |
| Python SDK | 将资源操作写成 Python 工作流 | 在需要可复用脚本时引入 |
| REST API | 直接通过 HTTP 集成，支持不同语言 | 需要细粒度控制时使用，不急于自行封装完整客户端 |
| 官方 Agent 技能 / 插件 | 提供平台操作指导和常用工作流 | 可以参考；是否安装或引入项目需要另行决定 |

官方当前快速入门通过 `pip install vastai` 安装 Python 包，SDK 示例使用 `from vastai import VastAI`。不同文档存在命名差异，实际集成时应固定所选版本并核对该版本接口。

## REST API 与认证

调研时官方文档记录的旧 base URL 为 `https://console.vast.ai/api/v0`，但实测实例列表的 v0 路径已经弃用。当前已验证的实例列表地址：

```text
https://console.vast.ai/api/v1/instances/
```

认证使用 `Authorization: Bearer <API_KEY>`。在 [Keys 页面](https://cloud.vast.ai/manage-keys/) 创建密钥。API key 用于管理平台账户和资源；SSH key 用于连接机器，它们不是同一种凭据。

只读鉴权检查示例；本项目已通过等价的 Python HTTP 请求验证该接口：

```bash
curl --fail-with-body --silent --show-error \
  -H "Authorization: Bearer $VAST_API_KEY" \
  https://console.vast.ai/api/v1/instances/
```

密钥从本地环境或凭据存储读取，不写入仓库、实验日志或对话。官方允许创建限定权限的 key，默认 key 可能具有完整账户权限。

## 可以自动化哪些步骤

1. 查询可租报价，按显卡、显存、价格、磁盘、网络等条件筛选。
2. 以报价 ID 创建实例，指定镜像、磁盘、SSH 模式与启动命令。
3. 保存返回的实例 ID，查询状态和连接信息。
4. 通过 SSH 执行环境检查、准备数据与训练命令，通过文件传输工具保存工件。
5. 查询账户与实例信息，在已有授权范围内停止或销毁资源。

**报价 ID 与实例 ID 不同。** 调研文档中的创建接口为 `PUT /api/v0/asks/{offer_id}/`，创建结果中的 `new_contract` 是后续操作需要的实例 ID。该创建接口没有实测，接入时需确认当前版本；不能根据实例列表的迁移推断所有接口均可直接替换为 v1。

平台 API 主要管理资源；训练脚本、训练进度与 checkpoint 的管理仍需 SSH 或我们自己的运行逻辑。实例显示 running 也不代表训练环境和任务已经通过检查。

## 自动化时需要处理的真实边界

- 创建实例开始产生存储费用；进入运行态后产生计算费用，数据传输也可能收费。
- 停止实例仍有存储费用，不能把 stop 当作费用完全归零。
- 销毁实例会删除其容器内数据，先转存需要保留的模型、日志与配置。独立持久卷等其他资源需单独核对生命周期。
- 等待启动应有超时，并处理 exited、unknown、offline 等异常状态，不能无限轮询。
- 创建失败或响应丢失时，先核对是否已成功创建，避免盲目重复创建付费资源。
- 自动化权限和平台能力是两回事。本项目当前只完成调研，不据此自动获得租卡或删除已有数据的授权；后续可事先约定预算、时长、资源范围与清理策略，减少重复确认。

## 推荐的项目接入顺序

本地凭据与实例列表只读验证已完成；下一步可验证报价查询。接着在明确预算与清理方式后完成一次小规模实例生命周期验证；最后再按实际需要加入项目脚本和技能。暂不安装官方插件、不引入完整资源编排系统。

## 本项目只读连接实测

- 从本地 `.env` 读取 `VAST_API_KEY`，没有输出密钥或完整账户响应。
- `GET /api/v0/instances/` 返回 HTTP 410，错误为 `deprecated_endpoint`，明确要求改用 `/api/v1/instances/`。
- `GET /api/v1/instances/` 返回 HTTP 200，实例列表为空，证明本次密钥鉴权和实例列表读取成功。
- 已确认 `.env` 被 Git 忽略；只提交不含凭据的记录。
- 该结果不证明创建、停止、销毁或其他接口的权限，也没有验证 CLI、SDK 和 SSH 连接。

经验：官方示例与线上版本可能不同。遇到 410 应先看经过脱敏的错误提示，不能直接断定密钥无效，也不应盲目重试或猜测所有接口版本。

## 自测

1. API key 和 SSH key 分别解决什么问题？
2. 为什么停止实例后仍可能收费，销毁之前又必须保存工件？
3. 为什么 API 返回 running，不能证明训练已经正常运行？

## 官方来源

- [API 概览](https://docs.vast.ai/api-reference/introduction)：接口定位、base URL 与认证。
- [API 完整入门](https://docs.vast.ai/api-reference/hello-world)：鉴权、搜索、创建、状态与连接流程。
- [CLI 入门](https://docs.vast.ai/cli/hello-world)：命令行操作、SSH key 与实例生命周期。
- [Python SDK 入门](https://docs.vast.ai/sdk/python/quickstart)：安装与调用示例。
- [API key 管理](https://docs.vast.ai/guides/reference/api-keys)：密钥创建、保存与权限。
- [Agent 集成](https://docs.vast.ai/guides/get-started/agents)：官方技能、插件与 JSON 输出。
- [实例计费](https://docs.vast.ai/guides/instances/pricing)：计算、存储与带宽费用。
- [实例管理](https://docs.vast.ai/guides/instances/manage-instances)：停止、销毁、状态与数据生命周期。
- [官方 CLI 仓库](https://github.com/vast-ai/vast-cli)：源码及 CLI / SDK 说明。

这类接口、包版本和价格相关信息可能变化，正式集成时复核。
