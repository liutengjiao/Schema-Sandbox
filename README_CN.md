# Schema Sandbox (图式沙箱)

[English](README.md) | [简体中文](README_CN.md)

**Schema Sandbox 是 AI Agent 安全执行的合同层。**

大语言模型是概率性生成的。生产环境下的 Agent 在调用工具、读写文件、访问 API、生成报告或触发业务流程之前，需要确定性的边界。

Schema Sandbox 定义了一套公共方法论，将模型输出转化为经过校验、授权、隔离和可审计的动作。它介于 LLM 推理与持久化 Agent 执行之间，强制执行输入合同、工具/API 语法、权限范围、工作空间分区、输出合同以及证据记录。

本仓库发布了 Schema Sandbox 的公开方法论草案、SIP-Core 规范说明、署名规则以及工业中间件参考模式。

> 状态: **公开方法学草案 v0.1.0**。本仓库仅提供方法论和规范参考，并非可直接用于生产的安全性运行时。

## 核心定位

提示词仅建议行为。  
MCP 仅连接工具。  
容器仅隔离代码。  
**Schema Sandbox 治理 Agent 执行合同。**

## 包含内容

- Schema Sandbox 公开方法论概述
- 基于清单（Manifest）的沙箱合同 SIP-Core 公开草案
- 用于清单、拒绝负载（Rejection Payload）和证据记录的 JSON Schema 示例
- 工业级 AI 中间件参考模式
- 署名与开源许可指南
- 公开学术引用元数据

## 不包含内容

- 生产环境运行时实现
- 官方认证服务
- 闭源或专有的工业级沙箱包
- 经过安全审计的工业级适配器
- 完整的基准测试复现包

## 仓库目录地图

```text
.
├── docs/                       # 方法论笔记、FAQ、署名指南、网站和 Substack 文案
├── specs/                      # SIP-Core 及相关公开规范草案
├── schemas/                    # JSON Schema 格式示例
├── examples/                   # 清单与数据负载参考示例
├── industrial/                 # 工业级 AI 中件间参考模式
├── papers/                     # 公开方法论论文草案（中/英文版）
├── assets/                     # Mermaid 格式的架构与流程图
├── release-notes/              # 公开版本发布日志
└── .github/                    # Issue 模板与 GitHub 元数据
```

## 工业中间件定位

Schema Sandbox 旨在介于 AI Agent 与企业或工业系统之间：

```text
AI Agent / LLM 编排器
        ↓
Schema Sandbox 运行时 API
        ↓
MES / ERP / QMS / CMMS / EHS / 数据库 / 工具 API
        ↓
校验后的输出 + 证据记录 (EvidenceRecord)
```

我们的目标不是替换已有的工业系统，而是通过输入合同、权限范围、工具调用授权、输出验证以及可审计的证据记录，让 AI Agent 能够安全地与这些系统进行交互。

## 快速示例

一个设备维护 Agent 提议创建一个高优先级的维护工单：

```json
{
  "agent_id": "maintenance_agent",
  "sandbox_id": "maintenance_workorder_gateway",
  "action": {
    "tool": "MES.createDraftWorkOrder",
    "args": {
      "machine_id": "CNC-07",
      "fault_type": "abnormal_vibration",
      "priority": "high"
    }
  }
}
```

Schema Sandbox 可能会拦截并返回：

```json
{
  "decision": "ask",
  "reason": "high_priority_work_order_requires_human_approval",
  "evidence_id": "ev_maintenance_001"
}
```

详见 [`examples/industrial-maintenance-workorder`](examples/industrial-maintenance-workorder/) 和 [`industrial/maintenance-workorder-gateway.md`](industrial/maintenance-workorder-gateway.md)。

## 许可证与商业边界

本仓库采用分层开源许可与商业保留模式。

- **SIP-Core 规范、文档、方法论文本和解释性材料** 采用 Creative Commons Attribution 4.0 International (**CC BY 4.0**) 许可，除非另有声明。参见 [`LICENSE-SPEC.md`](LICENSE-SPEC.md)。
- **源代码、JSON Schema、验证器、SDK、可执行示例、测试和示例 Manifest** 采用 **Apache License 2.0** 许可，除非另有声明。参见 [`LICENSE-CODE.md`](LICENSE-CODE.md)。
- **psi.run 托管运行时、Agent IP 平台、能力注册表、商业市场、商业 Sandbox Packs、私有 Prompt、评分系统、计费系统、托管 API、商标、品牌、认证标志和专有治理策略** 均不属于公开许可范围，除非另行书面同意。参见 [`COMMERCIAL_TERMS.md`](COMMERCIAL_TERMS.md)、[`TRADEMARK.md`](TRADEMARK.md) 和 [`CONFORMANCE.md`](CONFORMANCE.md)。

SIP-Core 作为将 Schema Sandbox 能力挂载到 Agent IP 上的公共极简接口合约发布。公开许可证允许开发者进行审查、独立实现和开发兼容工具，但不授予使用 psi.run 托管平台或声称获得官方认证、背书、商业市场上架或 psi.run 批准的权利。

参见 [`LICENSE.md`](LICENSE.md)、[`LICENSE-SPEC.md`](LICENSE-SPEC.md)、[`LICENSE-CODE.md`](LICENSE-CODE.md)、[`COMMERCIAL_TERMS.md`](COMMERCIAL_TERMS.md)、[`TRADEMARK.md`](TRADEMARK.md) 和 [`CONFORMANCE.md`](CONFORMANCE.md)。


## 推荐署名方式

> 基于 Tengjiao Liu 与 Hongzong Si 的 Schema Sandbox 方法论。

学术引用元数据可参见 [`CITATION.cff`](CITATION.cff)。

## 相关链接

- 官方网站: https://SchemaSandbox.com
- 代码仓库: https://github.com/liutengjiao/Schema-Sandbox
- 公开方法论大纲: [`docs/methodology-overview.md`](docs/methodology-overview.md)
- 工业级配置文件规范: [`specs/industrial-middleware-profile-v0.1.md`](specs/industrial-middleware-profile-v0.1.md)

## 免责声明

本仓库提供公开方法论和规范草案，不构成安全保证、法律意见或合规背书。在具有高风险的工业系统中使用本方法论，需要进行独立的工程、安全、合规与法律审查。
