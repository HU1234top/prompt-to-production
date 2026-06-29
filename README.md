[README.md](https://github.com/user-attachments/files/29454675/README.md)
# 🚀 From Prompt to Production：LLM Agent 技能从提示词到生产级

**让大模型多步骤技能在真实业务中可靠运行**

> "纯提示词约束无法实现 LLM Agent 多步技能的 99%+ 可靠执行。" —— 这是所有 Agent 框架面临的共性问题。

基于 [OpenClaw](https://github.com/openclaw/openclaw) AI Agent 框架 来自真实业务部署经验，我们总结了一套**分层防御框架**，将技能执行准确率从 ~70% 提升至 95%+。

---

## 🇨🇳 中文介绍

### 为什么这个问题重要

LLM Agent 擅长单次任务，但**多步骤业务技能**（涉及数据获取、计算、文档生成、系统写入）极其不可靠。我们观察到了三种失败模式：

| 失败模式 | 真实案例 |
|---|---|
| 🔴 **不读指令，自己编流程** | 用户触发技能，Agent 跳过 SKILL.md 自己瞎编 |
| 🟡 **读了但跳步** | Agent 读了指令但跳过关键验证步骤 |
| 🟠 **失败后过度发挥** | 执行失败后 Agent 反过来"审查"指令，列出7条修改建议，实际只有1条有问题 |

### 六层防御框架

每一层解决**不同环节**的问题，串联防护：

```
P3 斜杠命令     → 触发环节不再靠模型猜
  → P0 强制读取 → 确保技能内容被加载
    → P1 Few-shot → 通过示例提高遵从度
      → P2 结构化输出 → 执行结果可校验
        → P4 Checkpoint → 代码级防跳步
          → P5 子任务隔离 → 每步干净上下文
```

| 层级 | 做什么 | 成本 | 风险 | 回滚 |
|---|---|---|---|---|
| **P0** 强制读取 | `[CRITICAL]` 标记强制模型先读后执行 | 极低 | 极低 | 删标记 |
| **P1** 行为示例 | 真实执行案例（正确+反面）写入 SKILL.md | 低 | 低 | 删示例 |
| **P2** 结构化输出 | 每步 JSON Schema + 合理性校验 | 中 | 中 | 删Schema |
| **P3** 斜杠命令 | `/命令` 显式触发，绕过语义匹配 | 极低 | 极低 | 删配置 |
| **P4** 关键检查点 | Python 代码验证上一步完成才能执行下一步 | 中 | 中 | 删检查 |
| **P5** 子任务隔离 | 每步独立上下文，消灭注意力衰减 | 高 | 高 | 关隔离 |
| **P6** 框架注入 | 平台级强制注入技能内容（长期方向） | 高 | N/A | N/A |

### 核心洞察

1. **纯提示词有天花板** —— 无论怎么优化 Prompt，多轮对话后模型注意力都会衰减，这是当前 LLM 的架构限制
2. **每层只防自己的那一层** —— P0 防不住"读了但不照做"，但 P4 的代码检查能兜住跳步
3. **组合副作用可控但存在** —— P1 示例会跟 P0 抢注意力空间，但控制在 ≤1KB 就没问题
4. **自适应规则是静态约束的补充** —— 我们有一套错误驱动晋升机制：失败 → 记录 → 复发3次 → 晋升为硬约束

### 适用场景

- 为企业搭建 AI Agent 系统的 **AI 应用工程师**
- 设计多步骤 Agent 技能的 **Prompt 工程师**
- 评估 Agent 可靠性方案的 **工程管理者**
- 寻找真实故障模式的 **Agent 框架开发者**

---

## 🇬🇧 English Introduction

LLM Agents excel at single-shot tasks, but **multi-step business skills** (involving data fetching, calculations, document generation, and system writes) are notoriously unreliable. We've observed three failure modes:

| Failure Mode | Example |
|---|---|
| 🔴 **Doesn't read instructions, improvises** | Agent skips SKILL.md and makes up its own flow |
| 🟡 **Reads but skips steps** | Agent reads instructions but jumps ahead, skipping critical validation |
| 🟠 **Over-interprets on failure** | After failure, Agent "audits" instructions and suggests 7 fixes when only 1 was needed |

### The 6-Layer Defense Framework

Each layer solves a **different failure point**. They work in series:

| Layer | What It Does | Cost | Rollback |
|---|---|---|---|
| **P0** Enforced Read | `[CRITICAL]` markers force model to read before execution | Minimal | Delete markers |
| **P1** Few-shot Examples | Real execution examples (correct + counter) | Low | Delete examples |
| **P2** Structured Output | JSON schema per step with sanity validation | Medium | Remove schemas |
| **P3** Slash Commands | Explicit `/command` triggers | Minimal | Remove config |
| **P4** Checkpoints | Code-level step-skip prevention | Medium | Remove checks |
| **P5** Sub-agent Isolation | Fresh context per step | High | Disable spawn |
| **P6** Framework Injection | Platform-level forced injection (long-term) | High | N/A |

### Key Insights

1. **Pure Prompt Constraints Hit a Ceiling** — Attention drifts over long conversations; this is an architectural limitation
2. **Each Layer Has a Specific Failure Point** — P0 can't prevent "read but doesn't follow", but P4's checkpoint catches the skip
3. **Combination Side Effects Are Manageable** — Keep few-shot examples ≤1KB, only use P4 for high-risk steps
4. **Adaptive Error-Driven Rules Complement Static Constraints** — Frequently recurring errors auto-promote to hard constraints

### Real-World Context

This framework was developed during the deployment of an enterprise AI Agent platform that:

- Processes 20+ business requests daily with 50+ skills
- Covers 90%+ of core business workflows
- Has been running stably for 42+ days
- Reduced execution time from 2-3 hours to 3-5 minutes
- Improved accuracy from ~70% to 95%+

**Framework-agnostic** — applies to any LLM Agent system (LangChain, AutoGen, CrewAI, OpenClaw, etc.)

---

## 🗂️ 仓库结构 / Repository Structure

```
prompt-to-production/
├── README.md                          # 中英双语说明
├── docs/
│   └── skill-constraint-framework.md  # 完整框架文档（English）
├── images/                            # 架构图（规划中）
└── LICENSE                            # MIT License
```

## 🛤️ 路线图 / Roadmap

- [ ] 架构图（Mermaid）
- [ ] 真实案例研究（脱敏版）
- [ ] 代码模板实现指南
- [ ] 各层级效果 A/B 测试数据
- [ ] 中文版完整框架文档

## 🤝 参与贡献 / Contributing

如果你也遇到过类似的 Agent 可靠性挑战，欢迎分享你的解决方案！

## 📄 License

[MIT License](LICENSE) — Free to use, attribution appreciated.

---

<p align="center">
  <sub>Built with <a href="https://github.com/openclaw/openclaw">OpenClaw</a> · 从真实生产系统而来，不是 Demo</sub>
</p>
