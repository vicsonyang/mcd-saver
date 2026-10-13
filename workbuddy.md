# workbuddy.md — WorkBuddy 开发对话上下文

本文件记录本项目**使用 WorkBuddy 完成开发**的对话上下文，用于赛事核验是否符合联动活动奖励条件。

> **获取说明**：WorkBuddy 的官方文档目录中未见「导出对话」的独立说明页；社区资料提到对话页右上角提供导出入口（可选 TXT / Markdown / PDF）。**若你的界面中确实存在该入口，建议以官方导出结果替换或补充本文件**，本文件可作为规范化的上下文摘要保底。

---

## 一、会话概要

| 项 | 值 |
|---|---|
| 会话时间 | 2026-10-13（北京时间） |
| 任务类型 | 麦当劳程序员创意开发大赛参赛作品开发 |
| 工作模式 | Agent 模式（可读写文件、执行命令、调用连接器） |
| 产出仓库 | `mcd-saver`（麦麦省钱机器人） |
| 使用方 | 参赛者本人 |

---

## 二、开发过程：用户指令与 WorkBuddy 产出对照

| # | 用户指令（原文摘要） | WorkBuddy 的产出 |
|---|---|---|
| 1 | 提供赛事仓库链接，要求以 AI 为主完成参赛作品：理解赛题、制定方案、编写代码、整理提交材料 | 抓取并解析赛事主仓库四份文档（README / activityGuidelines / CONTEST_DECLARATION / RANKING）；产出《赛题分析与推进计划》文档 |
| 2 | 要求先分析赛题说明、评审标准和提交规范，明确交付物清单与关键时间节点，再给可执行计划，并说明需要用户配合的环节 | 明确「排名唯一依据为公开 Star 数」这一关键机制；列出 6 项交付物、4 个时间节点、7 项用户配合事项 |
| 3 | 「直接打包现有 mcd-saver，我有 GitHub 账号，但不知道怎么给你用」 | 确定走「打包现有技能」路线；读取本地技能全部资产；设计「双击运行、运行时交互询问账号」的推送脚本方案，**避免用户向我提供任何凭据** |
| 4 | （本轮） | 建立参赛仓库骨架；完成真实 MCP 只读链路验证；编写 README / MCP_INTEGRATION / mcp-config.example.json / CONTEST_DECLARATION / docs / LICENSE 等全部提交材料 |

---

## 三、WorkBuddy 能力实际使用记录

### 3.1 连接器调用（麦当劳 MCP）

开发过程中通过 WorkBuddy 的连接器能力**真实调用了麦当劳 MCP 服务**，用于验证接口口径与生成可复现的示例数据：

| Tool | 调用目的 | 是否只读 |
|---|---|---|
| `now-time-info` | 确认连接器可用、取服务器时间与星期 | 只读 |
| `query-nearby-stores` | 验证门店搜索与 `storeCode` 取得路径 | 只读 |
| `query-meals` | 验证菜单价格字段与单位口径 | 只读 |
| `query-store-coupons` | 验证门店可用券来源与返回结构 | 只读 |
| `calculate-price`（4 次） | 验证算价返回口径、候选对比、券挂载生效 | 只读 |

**本次开发未执行任何写操作**：未领券、未创建订单、未取消订单、未消耗积分。验证明细见 `docs/verification.md`。

### 3.2 本地文件与命令

- 读取本地已有技能资产（`SKILL.md`、8 个 `references/`、`scripts/check_environment.py`、`skill-dependencies.json`）
- 通过检索脚本统计技能实际引用的 MCP 工具清单（用于撰写 MCP_INTEGRATION.md）
- 建立仓库目录结构并迁移源码；生成带 CRLF 换行的 Windows 推送脚本
- 通过 GitHub 开放接口以原始格式获取官方 `CONTEST_DECLARATION.md`，**保证声明内容逐字一致**

### 3.3 网络与信息获取

- 解析赛事主仓库与麦当劳 MCP Server 仓库文档
- 查询 WorkBuddy 官方文档，确认连接器配置方式与本文件导出方式

---

## 四、关键判断与依据

| 判断 | 依据 |
|---|---|
| 排名策略重心应放在「README 可读性 + 传播」，而非堆技术 | 赛事规则明确排名**唯一依据是公开 Star 数**，无技术评审 |
| `mcp-config.example.json` 必须提交 | 主 README 表格漏列该文件，但权威规则文件 `activityGuidelines.md` 明确列为必需；按「多交不违规」处理 |
| 金额展示统一换算为元并保留两位小数 | 实测三套单位规则：菜单「元/字符串」、算价「分/整数」、`realDiscount`「元/数字」 |
| 价格字段只信服务端算价 | 本地估算会给出错误报价，是本类工具最严重的信任问题 |
| `CONTEST_DECLARATION.md` 必须逐字一致 | 赛事规则写明「文件名及内容均不可改动」 |
| 不在仓库中存放真实 Token | 赛事《信息安全声明》要求配置文件仅使用环境变量占位符 |

---

## 五、环境信息（脱敏）

| 项 | 值 |
|---|---|
| 操作系统 | Windows |
| Shell | Git Bash |
| Python | 3.13 |
| 连接器 | `mcd-mcp`（麦当劳中国 MCP Server，Streamable HTTP） |
| 凭据处理 | MCP Token 保存在使用者本机 WorkBuddy 连接器配置中，**未进入本仓库、未出现在任何对话输出或提交材料中** |

---

## 六、复现本项目的开发路径

1. 申请麦当劳 MCP Token 并配置 `mcd-mcp` 连接器（见 `README.md` 第三节）
2. 将本仓库放入 WorkBuddy 技能目录
3. 在对话中直接发起请求，例如「帮我点最便宜的」「这家店怎么点最省」
4. 环境自检：`python scripts/check_environment.py --capability 最优组合试算 --probe-results ./probe-results.json`
