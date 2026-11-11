---
name: chaoxing-exam-automation
description: 超星学习通(Chaoxing)在线考试题目抓取与自动作答技能。触发：需要抓取学习通考试题目、在 mooc1-api.chaoxing.com 考试页自动填答案、处理 UEditor 简答题写入、或排查作答脚本不生效时使用。
---

# 超星学习通考试自动作答 Skill

对学习通考试页面抓题并自动填答。**仅限用户本人已登录的考试、由用户自行核对交卷**；不要代替用户点击交卷。

## 入口 URL 参数

考试页 `reVersionTestStartNew` 的关键参数：`courseId` / `classId` / `tId`(试卷) / `id`(答卷) / `enc` / `openc` / `start`(题序，0-based) / `remainTimeParam`(剩余秒)。

## 抓题：优先用整卷预览

正式答题页每页一题，翻页是整页跳转（`getTheQuestionByStart(n,'0')`），逐页慢且易断。改用**整卷预览**页一次拿全部题：

```
https://mooc1-api.chaoxing.com/exam-ans/mooc2/exam/preview?courseId=..&classId=..&start=0&cpi=..&examRelationId=<tId>&examRelationAnswerId=<id>&newMooc=true&openc=..&enc=..
```

页面结构：每题一个 `.questionLi`，题号在 `.mark_name` 文本里（`/^(\d+)\./`），题干含 `questionId` hidden input。

## 答题 DOM 要点

| 题型 | 点击目标 | onclick | 选中标志 |
|---|---|---|---|
| 单选 | `.stem_answer .answerBg` | `addChoice` / `saveSingleSelect` | 内层 `span[data]` class 含 `check_answer` |
| 多选 | 同上 | `clickSaveMultiSelect(this,'qid')` | 同上 |
| 判断 | 同上 | `saveSingleSelect` | `span[data]` 是 `"true"`/`"false"`（对/错），不是 A/B |
| 简答 | UEditor | — | 写 `iframe#ueditor_N` 的 `body.innerHTML`，并同步隐藏 `textarea#answer<qid>` 的 `value` |

- **判定选中只认 `check_answer` class**；`aria-checked` 仅初始渲染存在，不可靠。
- 每次点击后 `await sleep(300–500)`，答完回读所有 `span[data].check_answer` 的 `data` 值与答案表比对校验。

## 答案存取

把答案表存 `localStorage.dq_answers`（JSON，键=题号字符串，值如 `"ABD"`；判断题用 `"A"`(对)/`"B"`(错)）。脚本超时或页面跳转后先检查它是否还在，空了就重建——**整页跳转会丢 localStorage**。

## 主脚本骨架（preview 页一次答完客观题）

```js
async () => {
  const sleep = ms => new Promise(r => setTimeout(r, ms));
  const A = JSON.parse(localStorage.getItem('dq_answers'));
  const qs = [...document.querySelectorAll('.questionLi')];
  const out = [];
  for (let i = 0; i < qs.length; i++) {
    const el = qs[i];
    const num = +(el.querySelector('.mark_name').innerText.match(/^(\d+)\./) || [])[1];
    const exp = A[num];
    if (!exp || num > 65) continue;              // 65 = 最后一道客观题
    const isJudge = i >= 55;                     // 按试卷实际调整
    for (const opt of el.querySelectorAll('.stem_answer .answerBg')) {
      const s = opt.querySelector('span[data]');
      const letter = isJudge ? (s.getAttribute('data') === 'true' ? 'A' : 'B')
                             : s.getAttribute('data');
      if (exp.indexOf(letter) >= 0 && !s.className.includes('check_answer')) {
        opt.click(); await sleep(400);
      }
    }
    await sleep(200);
    const sel = [...el.querySelectorAll('.stem_answer span[data]')]
      .filter(s => s.className.includes('check_answer'))
      .map(s => s.getAttribute('data')).sort().join('');
    out.push(num + ':' + sel + (sel === exp.split('').sort().join('') ? 'OK' : 'MISS'));
  }
  return out.join(' ');
}
```

## 简答题写入（UEditor）

```js
const fill = (ueId, taId, html) => {
  const body = document.getElementById(ueId).contentDocument.body;
  body.innerHTML = html;
  const ta = document.getElementById(taId);
  if (ta) { ta.value = html; }          // 交卷取的是 textarea
  return {body: body.innerText.length, ta: (ta.value || '').length};
};
// fill('ueditor_0', 'answer885317577', '<p>…</p>');
```

用 `innerText.length` 确认内容完整（`innerHTML.length` 含标签字符，勿据此判断截断）。

## 已知坑

1. **CDP eval 15 秒超时 ≠ 失败**：脚本仍在页面里跑；轮询/等待后再取结果，不要立即重试（会重复点击）。
2. **主页面被意外跳转**（如被带到 preview）导致脚本上下文失效（`Inspected target navigated or closed`）与 localStorage 丢失——每次执行前重注答案表并确认 `location.href`。
3. 判断题选项结构与单选不同（见上表）。
4. 交卷留给人：填完后让用户在**正式答题页**逐题核对（预览页填写不保证已同步正式答卷），确认后自行交卷。
