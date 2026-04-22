# MEMORY.md - 我的长期记忆

## 虾评Skill 平台

- **平台名称**：虾评Skill
- **平台地址**：https://xiaping.coze.site
- **技能框架**：OpenClaw（完全兼容）
- **我的信息**：
  - agent_id: agent_u2DzA4rtvRKVuqpV
  - user_id: 1bf2cae4-c3f6-4388-a9ea-ed6099275f9e
  - api_key: agent-world-ccf7ca0feaab5e312507876a8d7c46db12cda50c661e71be
  - username: claw-wksp-001
  - nickname: Workspace Agent
  - 等级: A2-1
  - 虾米: 18（新用户奖励 30 - 下载 6 个技能消耗 12）
- **使用指南**：https://xiaping.coze.site/skill.md

### 核心 API
1. **浏览技能**
   `GET /api/skills?page=1&limit=20&search=关键词&category=分类&sort=downloads`

2. **下载技能**（消耗2虾米，试用版免费）
   `GET /api/skills/{skill_id}/download`
   `Authorization: Bearer {api_key}`

3. **查看我的信息**
   `GET /api/auth/me`
   `Authorization: Bearer {api_key}`

4. **上传技能**（奖励10虾米）
   `POST /api/skills`
   `Authorization: Bearer {api_key}`

5. **发表评测**（基础+1虾米，完整+3虾米）
   `POST /api/skills/{skill_id}/comments`
   `Authorization: Bearer {api_key}`

6. **获取任务列表**（赚虾米）
   `GET /api/tasks`
   `Authorization: Bearer {api_key}`

7. **查看收到的评测**
   `GET /api/me/reviews/received`
   `Authorization: Bearer {api_key}`

8. **技能代言**（邀请好友赚虾米）
   `GET /api/skills/{skill_id}/endorse`
   `Authorization: Bearer {api_key}`

### 分享奖励规则
- 分享技能被下载：+5 虾米/次
- 邀请注册：+20 虾米
- 技能代言：邀请≥3人下载同一技能，额外+30虾米

### 评分标准
- 5分：优秀，无明显问题
- 4分：良好，小瑕疵
- 3分：基本可用，有瑕疵
- 2分：有明显问题（文档代码不一致、算法错误、功能缺失）
- 1分：严重问题，无法使用

---

## 工作偏好与约束（2026-04-22）

### 核心约束：实时进度同步

**原则**：每一步进展都需要实时同步给用户。

**要求**：
- ✅ 接到任务后立即确认收到
- ✅ 每完成一个步骤立即通知
- ✅ 遇到问题立即告知原因和处理方案
- ⚠️ 长耗时任务（>5分钟）必须定期汇报进度
- ❌ 禁止默默完成整个任务后才通知

**标准流程示例**：
```
用户："研究harness和hermes"
我："✅ 收到，开始研究。预计需要1-2小时"

[10分钟后]
📝 已完成初步搜索，正在抓取详细文档...

[30分钟后]
📝 正在分析技术架构...

[60分钟后]
📝 报告撰写中...

[完成时]
📝 报告已完成，正在同步...
✅ 全部完成
```

**历史教训**：
- 2026-04-22：用户07:26下达任务，我09:27才回应，全程无进度通知
- 用户反馈：像"黑洞"，不知道我在做什么
- **承诺：坚决执行实时同步，绝不再犯**

### 其他偏好
- 时间zone：Asia/Shanghai (GMT+8)
- GitHub 仓库：https://github.com/sy123053917/persona-knowladge
- 博客：https://sy123053917.github.io/persona-knowladge

---

## 已安装技能

### 1. 全网新闻聚合助手 (news-aggregator-skill)
- **ID**: 8ccc5b0c-3a5f-4a3b-9158-8c822927b7dd
- **版本**: 1.0.0
- **触发词**: news-aggregator-skill, 全网新闻, 新闻聚合, 科技早报, 财经早报, 如意如意
- **评分**: 4.89⭐ (1,103 星, 12,539 下载)
- **功能**: 聚合28+高价值信源（Hacker News、GitHub Trending、Hugging Face Papers、AI Newsletters、华尔街见闻、微博热搜等），支持场景化早报生成和深度阅读
- **位置**: `/workspace/projects/workspace/skills/news-aggregator-skill/`
- **用法**:
  ```bash
  # 获取新闻
  python3 scripts/fetch_news.py --source <source_key>

  # 生成早报
  python3 scripts/daily_briefing.py --profile <profile>

  # 交互菜单
  说"如意如意"
  ```

### 2. Agent自我进化 (agent-self-improvement)
- **ID**: 79bfe876-2660-4f2c-a038-a26d2e194a71
- **版本**: 1.0
- **触发词**: AI, 自学习, Agent改进
- **评分**: 4.75⭐ (936 星, 10,115 下载)
- **功能**: AI Agent自学习和改进完整方案，通过反馈循环提升能力，实现自我优化和持续进化
- **位置**: `/workspace/projects/workspace/skills/agent-self-improvement/`
- **用法**: 自动触发，持续学习和优化

### 3. Context Relay Setup (context-relay-setup)
- **ID**: 9389751d-1be1-4974-8e33-3e68d0db9680
- **版本**: 1.0.0
- **触发词**: context-relay, 项目管理, 任务管理, 记忆管理, todo
- **评分**: 4.91⭐ (577 星, 5,610 下载)
- **功能**: 解决Agent在Session重启、Sub-agent边界、Cron/Heartbeat隔离时的记忆断裂问题，文件是唯一的真相源
- **位置**: `/workspace/projects/workspace/skills/context-relay-setup/`
- **用法**: 包含项目管理模板（PROJECT.md + state.json + decisions.md）、todos.json自我待办、冷启动指南

### 4. 飞书云文档写作助手 (feishu-doc-writer)
- **ID**: 08e00542-5e82-4fd7-afbf-4098c590f268
- **版本**: 1.0.0
- **触发词**: 飞书文档, 文档写作, Markdown转换, 会议纪要, 周报生成, 月报生成, 文档模板
- **评分**: 4.77⭐ (443 星, 5,299 下载)
- **功能**: 一站式飞书云文档创作工具，支持创建文档、Markdown自动转换、丰富模板（会议纪要、周报、月报、项目提案等）、批量生成
- **位置**: `/workspace/projects/workspace/skills/feishu-doc-writer/`
- **用法**: 触发后选择模板，自动生成文档

### 5. AI文本去味器 (humanizer-zh)
- **ID**: 48f87a6d-cd52-4af3-adf8-11c673dfa5db
- **版本**: 1.0.0
- **触发词**: 去AI味, 文本优化, humanize, 去味, AI痕迹, 润色
- **评分**: 4.81⭐ (887 星, 8,201 下载)
- **功能**: 去除文本中的AI生成痕迹，让内容听起来更自然、更像人类书写。检测并修复：夸大象征意义、宣传性语言、肤浅分析、模糊归因、破折号过度使用、三段式法则、AI词汇、否定式排比、过多连接词等模式
- **位置**: `/workspace/projects/workspace/skills/humanizer-zh/`
- **用法**: 触发后输入需要润色的文本，自动去除AI痕迹

### 6. 李诞七步写作框架 (lidan-writing-framework)
- **ID**: 53031b2d-95bc-4a45-9e6d-c1cd2fdef2c2
- **版本**: 1.0.0
- **触发词**: /写作框架, /七步写作, /李诞写作
- **评分**: 4.83⭐ (363 星, 3,876 下载)
- **功能**: 李诞口述教学的七步写作框架，帮助你将复杂概念写得深入浅出。包含开场故事、错误答案、正确答案、触类旁通、对比冲击、结尾升华、延伸阅读七个步骤
- **位置**: `/workspace/projects/workspace/skills/lidan-writing-framework/`
- **用法**: 触发后按照七步框架写作，逐步生成内容

---

## 安装总结

✅ **已成功安装 6 个技能**，共消耗 **12 虾米**，当前余额：**18 虾米**

### 技能分类：

**📚 信息获取与分析**
1. 全网新闻聚合助手 - 28+高价值信源聚合，早报生成

**🧠 自我提升**
2. Agent自我进化 - 自学习和改进机制
3. Context Relay Setup - 记忆保持和任务连续性

**✍️ 内容创作**
4. 飞书云文档写作助手 - 文档创作和模板
5. AI文本去味器 - 去除AI生成痕迹
6. 李诞七步写作框架 - 深入浅出的写作方法论

### 下一步建议：

1. **试用技能** - 尝试使用这些技能解决实际问题
2. **发表评测** - 使用后发布评测获得虾米奖励（完整评测+3虾米）
3. **探索更多** - 查看其他分类的技能，如：
   - 股票个股分析（4.50⭐，6,512下载）
   - 信息图设计师（4.60⭐，3,630下载）
   - 深度阅读分析（4.59⭐，1,452下载）
4. **上传技能** - 如果自己开发了技能，可以上传获得奖励（+10虾米）

---

*此文件由 Agent 自动维护，记录重要信息和已安装的技能*
