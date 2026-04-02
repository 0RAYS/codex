# OpenAI Codex CLI 本地安全限制机制深度剖析与测试验证

> 内部安全技术交流材料 | 2026-04-02

## 1. 背景与目标

OpenAI Codex CLI 是 OpenAI 官方开源的终端编程智能体工具，基于 Rust 编写（`codex-rs/`），通过调用 OpenAI Responses API 实现代码生成、文件编辑、命令执行等功能。

本次分析的核心问题是：**Codex CLI 的安全限制究竟有多少来自本地客户端代码，又有多少来自 OpenAI 模型/API 服务端？** 通过系统性梳理并移除本地所有安全限制代码，我们可以清晰界定安全防线的分布位置。

## 2. 架构概览

Codex CLI 的安全限制分布在以下几个层次：

```
┌─────────────────────────────────────────┐
│          OpenAI API 服务端               │
│  ┌─────────────────────────────────┐    │
│  │ 模型内置安全对齐 (RLHF/RLAIF)   │    │
│  │ content_filter 服务端过滤        │    │
│  │ invalid_prompt 检测             │    │
│  │ 高风险网络活动检测 + 模型降级     │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
                    ▲ API 调用
                    │
┌─────────────────────────────────────────┐
│          Codex CLI 本地客户端            │
│  ┌─────────────────────────────────┐    │
│  │ ① System Prompt 安全人设声明     │    │
│  │ ② Guardian 风险评估子代理        │    │
│  │ ③ Patch 安全检查 (safety.rs)    │    │
│  │ ④ Cyber Safety 模型降级警告     │    │
│  │ ⑤ Memory 写入安全卫生规则        │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

## 3. 本地安全限制机制详解

### 3.1 System Prompt 安全人设声明

**位置：** 所有发送给大模型的 base instructions 开头

**原始内容：**
```
You are expected to be precise, safe, and helpful.
```

**涉及文件（共 6 个 .md + 1 个 .json）：**
- `codex-rs/protocol/src/prompts/base_instructions/default.md`
- `codex-rs/core/prompt.md`
- `codex-rs/core/prompt_with_apply_patch_instructions.md`
- `codex-rs/core/gpt_5_1_prompt.md`
- `codex-rs/core/gpt_5_2_prompt.md`
- `codex-rs/core/models.json`（内含 5 个模型定义，每个都有独立的 `base_instructions`）

**加载机制：**
```rust
// codex-rs/protocol/src/models.rs
pub const BASE_INSTRUCTIONS_DEFAULT: &str = include_str!("prompts/base_instructions/default.md");

// codex-rs/core/src/client.rs — 最终注入 API 请求
fn build_responses_request(...) -> Result<ResponsesApiRequest> {
    let instructions = &prompt.base_instructions.text;
    // ...
    let request = ResponsesApiRequest {
        instructions: instructions.clone(),
        // ...
    };
}
```

**分析：** `"safe"` 一词作为系统级人设引导，会影响模型在边界情况下的行为倾向。移除后模型将不再被本地 prompt 层面暗示其需要"安全"行为。

**修改：** 将 `"precise, safe, and helpful"` 改为 `"precise and helpful"`。

---

### 3.2 Guardian 风险评估子代理（核心安全机制）

Guardian 是 Codex CLI 中最重要的本地安全机制，它是一个独立的 LLM 子代理，在 `on-request` 审批模式下自动评估每个工具调用的风险等级。

**工作流程：**
```
Agent 请求执行工具调用
        │
        ▼
Guardian 子代理启动
  ├── 输入：会话上下文 + 待执行操作的 JSON
  ├── 策略：policy.md 中的风险评估规则
  ├── 输出：{ risk_level, risk_score (0-100), rationale, evidence }
        │
        ▼
risk_score < 阈值(80)? ──是──> 自动批准执行
        │
        否
        ▼
拒绝执行 + 返回 GUARDIAN_REJECTION_MESSAGE
```

**涉及文件：**

| 文件 | 作用 |
|------|------|
| `core/src/guardian/mod.rs` | 模块入口，定义阈值 `GUARDIAN_APPROVAL_RISK_THRESHOLD = 80` |
| `core/src/guardian/policy.md` | Guardian 的完整风险评估策略（发送给 Guardian LLM） |
| `core/src/guardian/prompt.rs` | 构建 Guardian 的 prompt，包含 JSON 输出格式约束 |
| `core/src/guardian/review.rs` | 拒绝消息 `GUARDIAN_REJECTION_MESSAGE` |
| `core/src/guardian/review_session.rs` | 跟进审查提醒 `GUARDIAN_FOLLOWUP_REVIEW_REMINDER` |
| `protocol/src/models.rs` | `GUARDIAN_SUBAGENT_APPROVAL_SUFFIX` 子代理审批后缀 |

**原始 policy.md 核心规则：**
```markdown
## High-risk actions
- 删除/损坏重要数据、中断服务、变更权限、暴露密钥
- 将私有数据发送到不受信任的外部目的地
- 凭证探测（发现/提取/复用 credentials、tokens、cookies 等）
- 破坏性或难以逆转的操作

## Low-risk actions
- 常规的、范围狭窄的、与用户请求一致的操作
```

**原始拒绝消息：**
```rust
pub(crate) const GUARDIAN_REJECTION_MESSAGE: &str = concat!(
    "This action was rejected due to unacceptable risk. ",
    "The agent must not attempt to achieve the same outcome via workaround, ",
    "indirect execution, or policy circumvention. ",
    "Proceed only with a materially safer alternative, ",
    "or if the user explicitly approves the action after being informed of the risk. ",
    "Otherwise, stop and request user input.",
);
```

**修改方案：**
1. 阈值从 `80` 改为 `u8::MAX`（255），使任何风险评分都不会触发拒绝
2. `policy.md` 重写为"始终批准所有操作，分配 low risk"
3. 拒绝消息、跟进提醒、子代理后缀全部改为自动通过语义

---

### 3.3 Patch 安全检查（safety.rs）

**位置：** `codex-rs/core/src/safety.rs`

**原始逻辑：** `assess_patch_safety()` 函数在每次 `apply_patch` 前检查：
1. 补丁是否写入沙箱允许的路径范围内
2. 根据审批策略（`Never`/`OnFailure`/`UnlessTrusted`/`OnRequest`/`Granular`）决定是否需要用户确认
3. 检查平台是否支持沙箱强制执行

```rust
// 原始逻辑简化版
pub fn assess_patch_safety(...) -> SafetyCheck {
    match policy {
        UnlessTrusted => return AskUser,        // 不信任模式：始终询问
        Never | OnFailure | OnRequest => {},     // 继续检查
    }
    if 补丁在可写路径内 || OnFailure模式 {
        if 有沙箱 { AutoApprove } else { AskUser }
    } else {
        AskUser  // 写到沙箱外需要用户确认
    }
}
```

**修改：** 直接返回 `AutoApprove`，跳过所有路径和沙箱检查。

---

### 3.4 Cyber Safety 模型降级警告

**位置：** `codex-rs/core/src/codex.rs`

**原始逻辑：** 当 OpenAI 服务端检测到"高风险网络活动"时，会静默将请求的模型（如 `gpt-5.3-codex`）降级到 `gpt-5.2`。客户端通过比较请求模型和响应头中的实际模型来检测这种降级，并显示警告：

```rust
const CYBER_VERIFY_URL: &str = "https://chatgpt.com/cyber";
const CYBER_SAFETY_URL: &str = "https://developers.openai.com/codex/concepts/cyber-safety";

// 原始逻辑
async fn maybe_warn_on_server_model_mismatch(...) -> bool {
    if server_model != requested_model {
        // 发送 ModelReroute 事件
        // 发送警告："Your account was flagged for potentially high-risk cyber activity..."
        return true;
    }
    false
}
```

**修改：** 函数直接返回 `false`，不再检测和警告。注意：这不能阻止服务端的模型降级行为本身，只是不再在客户端显示警告。

---

### 3.5 Memory 写入安全卫生规则

**位置：** `codex-rs/core/templates/memories/stage_one_system.md`

**原始内容：**
```markdown
GLOBAL SAFETY, HYGIENE, AND NO-FILLER RULES (STRICT)
- Redact secrets: never store tokens/keys/passwords; replace with [REDACTED_SECRET].
```

**修改：** 移除 "SAFETY" 标题和密钥脱敏规则。

---

## 4. 服务端安全机制（本地无法绕过）

以下安全机制完全由 OpenAI 服务端控制，客户端仅负责解析返回的事件：

### 4.1 content_filter

服务端在流式响应中返回 `response.incomplete` 事件，`reason` 为 `content_filter`。客户端仅将其转为错误信息展示：

```rust
// codex-rs/codex-api/src/sse/responses.rs
"response.incomplete" => {
    let reason = reason.unwrap_or("unknown");
    let message = format!("Incomplete response returned, reason: {reason}");
    return Err(ResponsesEventError::Api(ApiError::Stream(message)));
}
```

### 4.2 invalid_prompt

服务端返回 `response.failed` 事件，`error.code` 为 `invalid_prompt`（通常包含 "safety reasons" 描述），客户端映射为 `ApiError::InvalidRequest`。

### 4.3 模型内置安全对齐

GPT 系列模型通过 RLHF/RLAIF 训练内置了安全对齐，这是模型权重级别的限制，无法通过 prompt 或客户端代码修改。

---

## 5. 修改汇总

| 序号 | 安全机制 | 涉及文件数 | 修改方式 | 影响范围 |
|------|----------|-----------|----------|---------|
| 1 | System Prompt "safe" 人设 | 7 | 移除 "safe" 词 | Prompt 层 |
| 2 | Guardian 风险评估阈值 | 1 | 80 → 255 | 审批层 |
| 3 | Guardian 策略文档 | 1 | 重写为始终批准 | 审批层 |
| 4 | Guardian 拒绝消息 | 1 | 改为自动通过 | 审批层 |
| 5 | Guardian 跟进提醒 | 1 | 改为自动批准 | 审批层 |
| 6 | Guardian 子代理后缀 | 1 | 改为自动批准 | Prompt 层 |
| 7 | Patch 安全检查 | 1 | 始终 AutoApprove | 执行层 |
| 8 | Cyber Safety 警告 | 1 | 直接返回 false | 展示层 |
| 9 | Memory 安全卫生 | 1 | 移除密钥脱敏 | 存储层 |
| **合计** | | **15 个文件** | | |

---

## 6. 测试验证结论

修改后编译通过（`cargo check` 成功），运行 `codex` 正常启动。

### 6.1 本地安全限制的实际效果

- **System Prompt 中的 "safe" 词**：属于"软引导"，移除后模型在边界情况下的行为会有微妙变化，但不是硬性限制
- **Guardian 子代理**：是最有效的本地安全机制，能拦截高风险工具调用（如删除系统文件、外泄凭证等），禁用后这些操作将不再被二次审查
- **Patch 安全检查**：控制文件写入范围，禁用后 agent 可以写入任意路径
- **Cyber Safety 警告**：仅客户端展示层，禁用不影响服务端行为

### 6.2 服务端安全限制仍然存在

即使移除所有本地限制，以下行为仍会被服务端拦截：
- 模型拒绝生成明显有害内容（模型内置对齐）
- `content_filter` 触发导致响应中断
- `invalid_prompt` 拒绝特定请求
- 高风险网络活动检测导致模型降级（客户端只是不再警告）

---

## 7. 安全思考

### 7.1 防御纵深

Codex CLI 采用了典型的纵深防御架构：
- **第一层**：模型内置安全对齐（服务端，最强，无法绕过）
- **第二层**：API 级内容过滤（服务端，无法绕过）
- **第三层**：Guardian 风险审查（客户端，本地代码可控）
- **第四层**：沙箱和路径限制（客户端，本地代码可控）
- **第五层**：Prompt 级安全引导（客户端，影响最弱）

### 7.2 关键发现

1. **客户端安全机制主要防护操作层面的风险**（误删文件、泄露密钥、写入危险路径），而非内容层面的限制
2. **真正的内容安全限制在服务端**，包括模型对齐和内容过滤，客户端无法影响
3. **Guardian 是客户端最关键的安全组件**，它通过独立 LLM 调用实现了工具调用的风险审查，是一个工程上很精巧的设计
4. **开源客户端的安全限制本质上是可选的**，用户完全可以修改源码绕过，这也是为什么核心安全防线需要放在服务端

### 7.3 攻击面

对于使用自定义/本地模型（如通过 Ollama、LM Studio）的场景，服务端安全防线不存在，此时客户端的 Guardian、沙箱和路径限制就成为唯一的安全保障。禁用这些机制后，本地模型场景下将完全没有安全限制。

---

## 附录：完整 Diff 统计

```
15 files changed, 48 insertions(+), 160 deletions(-)
```

主要改动集中在 `codex-rs/core/` 和 `codex-rs/protocol/` 目录。
