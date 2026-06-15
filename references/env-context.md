# Environment Context for n8n Goals

## 用途

生成 n8n goal 的"环境上下文"字段时参考本文档。区分本地与云端两类部署环境，以及各自的接入方式差异，使 goal 中声明的环境信息通用、可移植、不绑定具体品牌或私有实例。

环境管理概念摘自开源项目 **n8n-as-code** by EtienneLescot（MIT License，https://github.com/EtienneLescot/n8n-as-code）。

---

## 1. 两类部署环境

n8n 实例只有两种物理落点，goal 中必须二选一声明：

| 类别 | 含义 | 标识词 |
|---|---|---|
| **本地部署（Local）** | 运行在操作者自己的机器或私有服务器上，常见为容器编排（Docker / Compose）或宿主机直装。实例归操作者全权管理。 | `local-docker` / `local-bare` |
| **云端部署（Cloud）** | 运行在 SaaS 或托管平台上，操作者只有 API 访问权，无操作系统级访问。通过 base-url + API key 接入。 | `cloud-managed` |

本地部署再细分为：
- **本地 Docker**：容器化部署，生命周期由本地实例管理器统一编排（启停、隧道）。
- **本地直装（bare）**：直接安装在宿主机或本地进程，无容器隔离层。

---

## 2. 接入方式差异

| 维度 | 本地部署 | 云端部署 |
|---|---|---|
| 主接入方式 | 直接 API 调用或本地实例管理器编排 | 仅 API（base-url + API key） |
| 文件系统访问 | 可直接读写本地 fs、挂载卷 | 通常禁用 fs，文件传输需经 SCP / API 上传 |
| 凭证存储 | 机器本地，不进仓库 | 平台侧托管 |
| 部分原生库 | 通常可用 | 云端可能禁用部分库或受限沙箱 |
| 网络出口 | 本机网络，可访问内网资源 | 受平台出口策略约束 |

写 goal 时按实际接入方式声明，不要假设云端有 fs 权限。

---

## 3. 环境管理概念

### Workspace Environment（仓库级环境）

- 一个仓库可挂载多个环境（如 dev、staging、prod），每个环境绑定一个实例地址 + 工作流目录。
- 环境是仓库上下文，而非机器资源：实例地址、API 凭证、工作流存放路径都在环境配置里。
- 同一时刻只有一个"当前环境"（active env），后续所有操作默认指向它。

### GitOps 同步（显式 pull / push）

- 同步是**显式**的：不被调用就不发生。不存在自动推送或自动拉取。
- `pull`：把远端工作流拉回本地源码（JSON / TypeScript 工作流文件）。
- `push`：把本地源码推到实例，提交前可 `--verify` 校验。
- 所有源码进版本控制，变更可 diff、可 review、可回滚。

### Promote（环境间提升）

- `promote --from <env> --to <env>`：把工作流源码从一个环境的目录搬到另一个环境的目录。
- 提升过程会重写目标项目元数据、重映射凭证与 Execute Workflow 引用、记录源到目标绑定。
- 默认推送，`--no-push` 可只迁移不推送；`--dry-run` 只规划不写盘。

---

## 4. 本地 vs 云端能力差异

云部署常受限，goal 中涉及文件、库、网络的操作必须适配：

- **文件系统**：云端通常禁用 `fs` 节点或限制写路径；需文件输入输出时改用 API 上传 / 临时存储 / 外部对象存储。
- **原生库 / 子进程**：云端可能禁用 `Execute Command`、部分 npm 依赖；本地部署通常无此限制。
- **触发器**：Webhook 在云端用平台域名；本地需经隧道暴露。
- **环境变量 / 凭证**：本地可读宿主机环境变量；云端凭证由平台侧托管，通过引用调用。

> 环境适配的完整设计模式参见 n8n-design-patterns。

---

## 5. Goal 的"环境上下文"字段写法

生成 goal 时，环境上下文字段应声明以下**通用类别**，不写死品牌、不写死真实 URL：

```yaml
environment_context:
  deployment: local-docker | local-bare | cloud-managed   # 二选一/三选一
  access: api-direct | api-key-only                        # 接入方式
  base_url: <占位，不填真实地址>                              # 云端必填，本地可省
  fs_access: available | restricted                        # 文件系统能力
  data_sources:                                            # 数据源类别（通用）
    - 类型（如 sheet / database / api / object-storage）
  notifications:                                           # 通知渠道类别（通用）
    - 类型（如 im / email / webhook / queue）
  sync_model: explicit-pull-push                           # GitOps 同步策略
  credentials_location: machine-local | platform-managed   # 凭证存放
```

### 写字段时的硬约束

1. **只写通用类别**：用 `local-docker` 而非具体实例品牌；用 `api-key-only` 而非具体平台名。
2. **占位真实地址**：`base_url` 写占位符，不写真实 URL；凭证从不进 goal。
3. **能力声明而非工具声明**：声明"有 fs 权限"或"只能 API"，不绑定具体 CLI 或 SDK。
4. **环境数量按需**：dev/prod 等多环境只在确实多环境时声明；单环境不要硬凑。

---

## 概念出处

本文档的环境分类（本地 managed instance / 云端 base-url + API key）、workspace environment、显式 pull/push 同步、promote 提升机制，均摘自 n8n-as-code 的设计模型。

- 项目：n8n-as-code by EtienneLescot
- License：MIT
- 地址：https://github.com/EtienneLescot/n8n-as-code

本文档仅摘录其公开 README 中的环境管理概念，已去除所有人名、本地路径、真实 URL，保持开源纯净。
