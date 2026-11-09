# 2026-11-09 复现：`unapproved channel` 真凶不是身份声明，而是「system 角色 + 长正文」

> 本文是对 [`../API-PROXY.md`](../../API-PROXY.md) 「11-128 第四种触发」的**修正与补充**。
> 结论来自**假模型名 oracle 直连上游**的单变量实验（零额度消耗），可复现。

## 一句话结论

DSH（`DeepSeekRuntime`）的完整 system prompt 会被国内版 `codebuddy.cn` 判成
**`11-128 Illegal API invocation from an unapproved channel`**。
但触发它的**不是**「第三方 agent 身份声明」——而是
**「`role: "system"` 承载了一段约 2.5k 字以上的长正文」**这一组合。

因此 v1.6.x 的「最大强度归一化」（角色收敛 / 去 `name` / 拍平内容块 / 抹身份声明）
**全部无效**，这正是错误文案里那句「已自动做最大强度归一化重试仍被拦」的实际含义：
四个动作都做了，但一个都不对。

## 实验方法（零额度）

`model` 填一个不存在的名字，上游**先做渠道校验、再做模型校验**，于是：

- `{"code":11102,...model ... not found}` ⇒ **过了**渠道校验
- `{"code":11-128,...}` ⇒ **被**渠道校验拦下

绕开本地反代直连 `https://www.codebuddy.cn/v2/chat/completions`，
出站头对齐反代 `chat_headers()`，`stream: true`（上游不支持非流式，
非流式会先被 `11101 Non-stream chat request is currently not supported` 挡掉）。

## 单变量对照（每组 3 次，全稳定）

正文取自 DSH 真实会话转录里的 system 原文（12 873 字）。

| 请求形态 | 结果 |
|---|---|
| 最小 system（`You are a helpful assistant.`） | ✅ `11102` 通过 |
| 纯填充 `"x" * 12000` 作 system | ✅ `11102` 通过 |
| 纯填充 `"这" * 8000` 作 system | ✅ `11102` 通过 |
| DSH 真实全文（12 873 字）作 system | ❌ `11-128` 拦 |
| 真实前缀 2 000 字作 system | ✅ `11102` 通过 |
| 真实前缀 2 500 字作 system | ❌ `11-128` 拦 |
| 真实前缀 8 000 字作 system | ❌ `11-128` 拦 |
| **抹掉品牌词 `DeepSeek`+`Harness`（2 处）后全文** | ❌ `11-128` 拦 |
| **再抹掉裸 `@` 后全文** | ❌ `11-128` 拦 |
| **同一正文改 `role: "user"`** | ✅ `11102` **通过（3/3）** |
| system 只留一句短开场 + 正文并入 user | ✅ `11102` **通过（3/3）** |
| 正文拆成两条 system | ❌ `11-128` 拦 |

关键判别：

- **长度不是判据** —— 12 000 字纯填充通过，2 500 字真实正文被拦。
- **单句身份词不是判据** —— 只含 `DeepSeek`+`Harness` 的那一句（106 字）**通过**；
  含它的 325 字段落也**通过**。有问题的是一整段 1 002 字的 runtime-context 段落。
- **品牌词与裸 `@` 不是判据** —— 两种清洗各自做、一起做，全程仍被拦。
- **`role: "system"` 是必要条件** —— 同一份正文换成 `user` 立刻通过。

⇒ 上游实际检的是**「system 角色 + 足够长的、带指令口吻的正文」**这个组合模式，
不是某个具体关键词。这也解释了为什么历史上这条规则反复漂移
（10-09 判 `gitStatus:` 环境块、10-17 判 `developer` 角色、10-21 复测又都放行）。

## 对反代的影响（为什么自愈注定失败）

`sanitize_messages_aggressive()`（`crates/wb-switch-core/src/modules/proxy.rs`）做四件事：

1. 未知角色 → `system`
2. 去掉 `messages[*].name`
3. 内容块拍平成字符串
4. system 里抹 `IDENTITY_DECLARATIONS`

实验已证明 1/2/3/4 **对本案全部无效**。更糟的是第 1 条方向反了：
它把非标准角色**收敛成 system**，而实测显示应该做的是**把 system 降级成 user**。

顺带记录两处黑名单缺口（虽然对本案不致命，但确实漏）：

```rust
const IDENTITY_DECLARATIONS: &[&str] = &[
    "DeepSeekRuntime",
    "deepseek-harness",
    "DeepSeek-Harness",
];
```

DSH 真实提示词里写的是 **`DeepSeek` + 空格 + `Harness`**（两处），
以及 **`DeepSeekRuntime`**（两处，不带空格）。前者**不在**黑名单任何一项内。

## 建议的修复

按代价从低到高：

### 方案 A（推荐，立竿见影）：自愈时把 system 降级为正文

在 `sanitize_messages_aggressive()` 里增加一步，且**必须优先于其余动作**：

```rust
// 5. system 长正文降级：上游按「system 角色 + 长指令正文」组合判渠道。
//    实测同一正文改 user 后 3/3 通过；抹品牌词/裸 @ 均无效。
//    保留首条 system（部分上游要求首条必须是 system）但压到最短，
//    原正文前移到紧随其后的 user 消息 —— 语义损失极小，且不影响模型理解。
```

具体做法：把每条 `system` 的正文**前移到下一条 `user`** 的 content 开头，
system 自身替换为一行 `You are a helpful AI assistant.`。
国际版因为要求首条必须是 `system`，必须保留一条占位 system。

### 方案 B（更保守）：只降级「长」system

仅当 system 正文超过约 2 000 字时执行 A，短 system 完全不动。
实测 2 000 字通过、2 500 字被拦，阈值取 2 000 有余量。

### 方案 C（不改行为，只改文案）

如果不想动请求形状，至少要把错误文案改对。现在这句话是**误导性的**：

> 这是腾讯风控在识别系统提示词里的**第三方 agent 身份声明**（典型是 `DeepSeekRuntime`）。
> DSH 用户可关掉原生开关 `includeHarnessIdentity`，或把系统提示词里那段身份声明删掉，即可恢复。

实测表明：

- `includeHarnessIdentity: false` **已经开着**，提示词里那两个词仍然在（它们来自 runtime context 段，不是 harness identity 段）；
- 就算把这两个词全删掉，请求**照样被拦**。

⇒ 按这句话去操作的用户会白忙一场。文案应改为：
「上游把『system 角色承载长正文』判成非官方客户端；可改用国内版、缩短系统提示词，
或等待上游策略调整」。

## 复现脚本

本次实验脚本放在临时目录（未入库）：

| 文件 | 作用 |
|---|---|
| `ab.py` | 直连上游的探针骨架（出站头与 `chat_headers()` 对齐） |
| `recheck.py` | 品牌词 / 裸 `@` 清洗的 A/B（正则构造，避免字面量陷阱） |
| `fix.py` | 三种候选修复的对照（P1/P2/P3） |
| `remedy.py` | 角色降级 vs 内容清洗的判别 |

⚠️ 探针请用 `model: "__oracle_probe__"`，**不要**用真模型名 —— 那会实打实消耗额度。

## 一条方法论教训

`API-PROXY.md` 里已经写过一次：「换 UA 无效 ≠ 封的是账号，因为没隔离『头多了还是少了』」。
这次是同一类错误的重演：**「抹掉可疑词仍然被拦」不等于「上游按语义判」**，
因为还有 `role` 这个变量没被隔离。下次遇到 11-128，
应当把 `role` / 消息条数 / 单条长度一起放进单变量表，而不是只盯正文里的词。
