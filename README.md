# chaoxing-exam-automation

超星学习通（Chaoxing）在线考试抓题与自动作答——**Agent Skill 格式**（`SKILL.md`）。

## 用途

让通用 agent（Claude Skill 规范兼容）在用户本人已登录的学习通考试页里：

1. 通过「整卷预览」接口一次性抓取全部题目；
2. 按答案表自动点选客观题（单选/多选/判断）并逐题回读校验；
3. 向 UEditor 写入简答题答案并同步 textarea。

## 安装

把 `SKILL.md` 放入 agent 的技能目录（如 Claude Code 的 `~/.claude/skills/chaoxing-exam-automation/`）即可被自动发现。

## 免责声明

仅供学习自动化研究，请遵守所在学校/平台的考试规则；本工具不代替用户交卷，答案正确性由使用者自行核对。
