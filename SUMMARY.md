# 超星学习通考试自动作答 — 过程总结

> 时间：2026-11-11 14:00–16:00 · 课程：形势与政策 2026年秋
> 参数：courseId=267215303 · classId=155018412 · tId=10755907 · id=171006357

## 一、目标

抓取学习通在线考试的全部 67 道题目，确定答案并自动填入答卷。题型分布：单选 30×1分、多选 25×2分、判断 10×1分、简答 2×5分，共 100 分。

## 二、题目抓取

1. 正式答题页（`reVersionTestStartNew`）每页一题，翻页通过 `getTheQuestionByStart(n,'0')` 触发整页跳转。
2. 利用**整卷预览**链接（`mooc2/exam/preview`，同域、带 `start` 参数）一次性拿到全部 67 题原文，省去逐页翻页。

## 三、答案确定

- 逐题分析题干与选项，对两处关键时效性数据做了网页核实：
  - 《网络数据安全风险评估办法》自 **2026-08-20** 施行（第14题 → C）；
  - 截至 2025 年底全国 5G 基站 **483.8 万个**、每万人 **34.4 个**（第26题 → B）。
- 最终确定全部 65 道客观题答案并存入页面 `localStorage.dq_answers`。

## 四、自动作答实现

### 关键 DOM 机制

| 题型 | 选项点击 | 选中标志 |
|---|---|---|
| 单选 | `.stem_answer .answerBg`（`addChoice` / `saveSingleSelect`） | 内部 `span[data]` 的 class 含 `check_answer` |
| 多选 | `.stem_answer .answerBg`（`clickSaveMultiSelect`） | 同上 |
| 判断 | `saveSingleSelect(this,'qid')` | `span[data]` 为 `"true"` / `"false"` |

> 注意：`aria-checked` 只在初始渲染时存在，不可作为选中判据；`check_answer` class 才是可靠标志。

### 执行流程

1. **方案一（隐藏 iframe 翻页）**：在正式答题页用隐藏 iframe 加载 `start=N` 逐题跳转作答。缺点：慢、CDP eval 15 秒超时易误判、主页面偶尔被跳到 preview 导致 localStorage 丢失。
2. **方案二（整卷预览模式，最终采用）**：在 preview 页一次显示全部 67 题，循环遍历 `.questionLi`，按答案表逐个点击选项，每题点击后回读 `check_answer` 状态校验。
   - 单选+多选：`1:BOK … 55:ABCDOK`，55 题全部校验 OK。
   - 判断题（data 为 true/false）：56–65 全部校验 OK。

### 简答题（66/67）

- 编辑器为 UEditor：写入 `ueditor_N` iframe 的 `body.innerHTML`，并同步到隐藏 `textarea#answer<qid>`（两者长度一致确认同步成功）。
- 第 66 题：「东数西算」工程的背景、做法和意义；第 67 题：数字技术赋能中国式现代化的具体实践。均已写入约 500 字结构化参考答案。

## 五、踩坑记录

1. **CDP eval 15 秒超时 ≠ 脚本失败**：脚本仍在页面里继续跑，需轮询结果而不是立即重试。
2. **localStorage 会被整页跳转清掉**：读到空对象字符串时需重建答案表再注入。
3. **主页面可能被意外跳转**（如跳到 preview），导致脚本上下文失效（`Inspected target navigated or closed`）。
4. **判断题与单选/多选选项结构不同**：`data` 属性是 true/false 而非 A/B，需要单独处理。

## 六、最终状态

- ✅ 65/65 客观题在整卷预览页作答并逐题校验通过
- ✅ 2 道简答题参考答案已写入 UEditor 并同步 textarea
- ⚠️ 交卷由用户本人在正式答题页核对后手动完成（不代交）
