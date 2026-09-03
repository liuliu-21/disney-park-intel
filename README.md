# 🏰 Disney Park Intelligence

[English](README_EN.md) | 中文

一个为上海迪士尼做的 AI 行程规划器。

我做它的原因很简单：乐园攻略很多，但真正走进园区后，人需要的不是又一张“必玩榜单”，而是一条能同时考虑当前排队、步行距离、身高限制、餐厅预约、尊享卡和个人偏好的可执行路线。

👉 [在线体验](https://disney-park-intel.vercel.app)

## 它能做什么

- 根据同行人身高、偏好、节奏和尊享卡生成当日路线
- 结合实时排队、历史快照和步行成本安排项目
- 把餐厅预约、演出和“一定要去”当作硬约束
- 用自然语言调整行程，并实时展示 Agent 的工具调用
- 提供项目、餐厅、商店和拍照点信息，弱网时也能打开已缓存页面

## 我真正花时间的地方

这个项目最难的不是让页面跑起来，而是证明结果不只是“看起来合理”。

早期版本有过几个很隐蔽的问题：外部数据源 ID 没有真正对上内部项目；中文评论按空格分词，使检索得分几乎全是 0；步行成本一直从起点计算，导致行程在园区间来回跳。这些 bug 不会让系统报错，只会让它给出错误建议。

我补了确定性评测和边界测试，重写中文 BM25，统一数据源 ID，并重构行程排程。我想做的不只是一个 LLM 界面，而是一个尽量可测试、可解释，也会承认数据不足的决策系统。

## 它是怎么工作的

```text
用户偏好 + 时间/身高/预约约束
                    │
                    ▼
          候选项目过滤与评分
                    │
                    ▼
  实时等待 > 历史预测 > 静态基线
                    │
                    ▼
      贪心路由 + 硬锚点 + 空档填充
                    │
                    ▼
          Claude 生成简短解释
```

路由不会让 LLM 凭空安排。时间、身高、尊享卡间隔和预约锚点由确定性代码处理；Claude 负责选择工具、理解意图和解释结果。Agent 最多运行 5 轮，通过 SSE 流式返回。

```text
cost = waitWeight × effectiveWait
     + walkWeight × walkMinutes
     + energyWeight × thrillScore × 5
```

`efficient`、`balanced` 和 `easy` 三种模式会调整等待、步行和体力的权重。

## 数据与降级策略

| 能力 | 来源 | 不可用时 |
|---|---|---|
| 实时等待 | themeparks.wiki，24 个项目中映射 20 个 | 返回静态基线，标记 `fallback: true` |
| 等待预测 | 仓库中的历史快照 | 少于 8 个样本时用当前快照外推，信心为 low |
| 游玩评论 | 离线采集的小红书公开笔记 | 未覆盖项目使用明确标记的人工示例 |
| AI 评分 | Claude 结构化输出 | 无凭证或调用失败时使用本地规则 |
| 多轮会话 | 进程内存，可选 Upstash Redis | 无 Redis 时冷启动会丢失会话 |

当前数据包含 24 个游乐项目、11 家餐厅、29 家商店、44 个拍照机位；14 个项目的 280 条真实小红书笔记；以及 1,300 条版本化的排队快照。

这些数字不是“用户数”。项目目前还没有可验证的活跃用户或留存数据。

### 已知局限

- 小红书没有星级，`rating` 只是热度代理，不是满意度。
- 词典法不擅长处理口语、话题标签和 emoji，约 75% 的真实笔记被判为 neutral。
- 餐厅真实评论尚未采集，相关结果会显示降级标记。

## 可复现的结果

```bash
npm test
```

170 条测试，覆盖数据源 ID、等待时间、身高边界、尊享卡、路由约束、中文检索、会话持久化、限流和 SSE 分帧。3 条联网检查默认跳过。

```bash
RATE_LIMIT_LLM=100000 npm run dev
python3 scripts/eval_itinerary.py
```

**100/100 行程场景通过**，覆盖普通、时间、身高、尊享卡、锚点和路线模式。评分由确定性脚本完成，不调用 LLM。

```bash
npm run eval:retrieval
```

| P@1 | P@3 | Recall@3 | MRR | nDCG@5 |
|---:|---:|---:|---:|---:|
| 0.944 | 0.556 | 0.678 | 0.963 | 0.790 |

检索结果基于 14 条人工示例和 18 个单人标注查询，只适合比较算法改动，**不代表真实线上质量**。`eval_tool_accuracy.py` 尚未运行，仓库中不会在没有结果时填一个数字。

## 本地运行

```bash
git clone https://github.com/liuliu-21/disney-park-intel.git
cd disney-park-intel
npm install
cp .env.local.example .env.local
npm run dev
```

打开 [http://localhost:3000](http://localhost:3000)。不配置 Anthropic 凭证也能使用基础功能：评分会退回本地规则，AI 助手则会返回带原因的 503。

> 如果不用 `ANTHROPIC_API_KEY`，请删掉整行，不要留空值。空值仍会遮蔽其他可用凭证。

## 技术栈与目录

- Next.js 14 App Router、TypeScript、Tailwind CSS、Zustand
- Anthropic Tool Use API + SSE streaming
- BM25 + 中文字符二元组
- Vitest、GitHub Actions、Vercel，可选 Upstash Redis

```text
src/app/                 页面与 API routes
src/app/api/agent/       Agent 编排、工具定义与执行
src/lib/routing.ts       路径规划与约束
src/lib/wait-*.ts        实时等待与历史预测
src/lib/vector-store.ts  BM25 检索
data/                    评论与排队快照
scripts/                 采集、校验和评测
```

## 贡献与归属

仓库历史中的 `JINGYU LIU` 和 `Myla0619` 是同一位作者的两个 GitHub 身份。其余快照提交由该作者配置的 GitHub Actions 自动生成。

外部 API、迪士尼项目信息和小红书公开内容并非本项目原创；本项目实现了它们的采集、映射、降级处理、检索和评测。
