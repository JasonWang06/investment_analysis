# Three Engines 批量分析 — 执行方案 v2（实战优化版）

## 从 v1 学到的教训

| v1 设计 | 实际结果 | 结论 |
|---------|---------|------|
| 8个 Search Agent 并行 | 1个成功、1个卡 plan mode、1个未知 | ❌ Agent 不可靠 |
| 8个 Writer Agent 并行 | 3个全部超时（27分钟） | ❌ Agent 写报告太慢 |
| 主线程直接写报告 | 每只2-3分钟，零超时，100%成功 | ✅ 最优解 |
| 主线程并行 Web Search | 6个搜索同时发，~1分钟完成 | ✅ 效率高 |

## 核心原则

1. **不用 Agent** — 全部在主线程执行，零超时风险
2. **搜索并行化** — 每轮6个 Web Search 同时发（工具并行上限）
3. **写入立刻执行** — 搜完数据立刻写报告，不存中间 JSON（省一步）
4. **每次提交 ≤ 6只股票** — 控制上下文不溢出

---

## 执行架构

```
每次提交（6只股票，~15分钟）：
  Step 1: 6个 Web Search 并行（每只1个搜索）     → 1-2分钟
  Step 2: 直接写 Stock 1 报告                      → 2-3分钟
  Step 3: 直接写 Stock 2 报告                      → 2-3分钟
  Step 4: 直接写 Stock 3 报告                      → 2-3分钟
  Step 5: 直接写 Stock 4 报告                      → 2-3分钟
  Step 6: 直接写 Stock 5 报告                      → 2-3分钟
  Step 7: 直接写 Stock 6 报告                      → 2-3分钟
  Step 8: git add + commit + push                  → 30秒
```

---

## 提交模板（复制粘贴，改股票列表即可）

### 完整分析版（Batch 1-2，每次6只）

```
对以下6只股票做 Three Engines 完整分析。

⚠️ 执行规则：
- 先对6只股票各搜1次（并行），获取最新财务数据
- 然后逐只写报告到 ./reports/{TICKER}.md
- 每写完一只立刻写入文件，不要等全部完成
- 不要用 Agent，全部在主线程完成
- 写完全部后 git add + commit + push

股票列表：{NOW, AMZN, NFLX, ISRG, SHOP, GLBE}

用户能力圈（判断 Gate 0 用）：
- 圈一（最强）：AI SaaS、网络安全、云平台、大厂科技、消费应用
- 圈二（中等）：跨境电商（SHOP/GLBE）、消费零售
- 圈三（基础）：金融科技
- 不熟悉：保险精算、拉美市场、医药研发、传统银行
- 已持仓：CRWD, GOOGL, TSLA, RBLX, BOTZ

每只股票的报告格式（用中文写，保留英文ticker和财务术语）：

# {Company} ({TICKER}) — Three Engines Analysis
Date: 2026-04-09 | Price: ${X} | MCap: ${X}B

## Gate 0: Circle of Competence — PASS/FAIL
（2-3句话。FAIL则到此结束）

## Engine 1: Business Quality — X/25
5个维度各3-5句话+分数：
A. Moat (1-5)：转换成本、网络效应、规模、品牌、数据
B. Economics (1-5)：毛利率、利润率趋势、FCF、vs竞品
C. Growth (1-5)：有机vs收购、TAM、可预测性
D. Competition (1-5)：行业格局、份额方向、AI顺风/逆风
E. Antifragility (1-5)：衰退韧性、技术趋势、监管
总结表格 → Proceed(≥15) / Watchlist(10-14,停止) / Reject(<10,停止)

## Engine 2: Management Quality — X/20
4个维度各3-5句话+分数：
A. Capital Allocation (1-5)
B. Incentive Alignment (1-5)
C. Strategic Discipline (1-5)
D. Integrity (1-5)
一票否决检查 → Proceed(≥12) / Monitor(8-11,停止) / Reject(<8,停止)

## Engine 3: Valuation
估值表（P/E, P/S, P/FCF vs历史vs行业）→ 一句话历史位置
Base DCF：假设表 + Y1/Y2/Y3/Y5/Y10逐年 + 折现 + 每股公允价值
Bull/Bear各一句话+公允价值
安全边际 = (Fair - Price) / Fair → 达标(≥30%)/未达标
（E1=15-19时需≥50%）

## Final Signal
STRONG BUY(≤70%FV) / BUY(≤90%FV) / WATCH(>90%FV)
低估原因：Cat 1(系统性) / Cat 2(临时) / Cat 3(恶化)
决定：Execute / Override / Wait（一句话理由）

## Kill Criteria（仅BUY/STRONG BUY时写）
3个可量化的卖出条件
```

### 快速筛选版（Batch 3-4，每次15只）

```
对以下15只股票做快速筛选。

⚠️ 执行规则：
- 不做 web search，基于你的知识库判断
- 全部输出到一个文件 ./reports/BATCH{N}_SCREEN.md
- 每只5-8行，不需要完整分析
- 写完 git add + commit + push

股票：{ZS, FTNT, OKTA, CYBR, NET, CRM, SNOW, MDB, HUBS, WDAY, VEEV, PATH, BAH, AAPL, ORCL}

用户能力圈：[同上]

每只股票格式：
### {TICKER} — {Company} | ~${price}
- Gate 0: PASS/FAIL（一句话）
- E1快评: Moat X/5, Economics X/5, Growth X/5, Competition X/5, Anti X/5 = X/25
- 估值: 当前大致P/E → 贵/合理/便宜（基于你的知识）
- Signal: WATCH / NO SIGNAL / 值得完整分析
- 一句话理由
```

### Gate 0 淘汰版（Batch 4 能力圈外，一次可做全部）

```
对以下股票做 Gate 0 能力圈判定。不做分析。

⚠️ 执行规则：
- 不做 web search
- 全部输出到 ./reports/GATE0_ELIMINATED.md
- 每只2-3行
- 写完 git add + commit + push

用户能力圈：[同上]

股票列表：
HIG, KNSL, LMND, ROOT, PAGS, NU, MELI, LPRO,
SDGR, GMED, IRTC, TEM,
JPM, GS, MS, FIS, FISV, GPN,
MARA, RIOT, CLSK, MSTR, ETHE,
JOBY, LCID, RIVN, SOUN, BBAI,
WISH, CHGG, BEST, YSG, BZUN, BOTZ

格式：
### {TICKER} — {Company}
Gate 0: FAIL — {原因：能力圈外/已死/无护城河/ETF}
```

---

## 完整执行清单

| 提交 | 内容 | 股票数 | 预计时间 | 累计 |
|------|------|--------|---------|------|
| ✅ 已完成 | Batch 1 完整分析 | 8只 | — | 8 |
| ✅ 已完成 | Batch 2 部分 (PANW/MSFT/META) | 3只 | — | 11 |
| 提交1 | Batch 2 剩余完整分析 | 6只 | ~15分钟 | 17 |
| 提交2 | Batch 2 最后3只 | 3只 | ~10分钟 | 20 |
| 提交3 | Batch 3 快速筛选 (上半) | 15只 | ~10分钟 | 35 |
| 提交4 | Batch 3 快速筛选 (下半) | 15只 | ~10分钟 | 50 |
| 提交5 | Batch 4 Gate 0 淘汰 | ~35只 | ~5分钟 | 85 |
| 提交6 | Batch 4 剩余快速筛选 | ~50只 | ~15分钟 | ~135 |
| 提交7 | 汇总 SUMMARY.md | — | ~5分钟 | Done |
| **总计** | | **~135只** | **~70分钟** | |

## Troubleshooting

**如果某只股票 web search 失败**：跳过，用训练数据中的知识写报告，标注 "[数据基于2025年中，非最新]"

**如果一次提交中途被中断**：检查 ./reports/ 看已写完哪些，下次只写剩余的

**如果上下文快满了**：开新会话，从最新的提交点继续。报告都在文件里不会丢。

---

## 与 v1 的关键差异

| 维度 | v1 | v2 |
|------|----|----|
| 执行方式 | 并行 Agent | 主线程直接写 |
| 搜索 | Agent 各自搜 | 主线程并行搜索(6个/轮) |
| 写报告 | Agent 写 | 主线程直接写 |
| 中间存储 | JSON文件 | 不需要（搜完直接写） |
| 超时风险 | 高（实测100%超时） | 零 |
| 每只耗时 | 27分钟（超时） | 2-3分钟 |
| 可靠性 | ~30% | ~100% |
