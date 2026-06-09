# AI剪口播

一个**剪口播视频的 coding-agent skill**:自动转录 → 识别口误/口癖/静音 → 网页波形审核 → 导出 FCPXML,拖进剪映或 Final Cut Pro 完成最后一刀。核心是一份 [`SKILL.md`](SKILL.md),**任何能读文件、能跑 shell 的 coding agent 都能用**(Claude Code、Codex、Gemini CLI、Kimi 等);也可以直接装进 Claude Code 当 skill。

## 这个 Skill 做什么

口播视频最费时的不是剪,是**找哪里该删** —— 重复重说、卡顿、口癖、长停顿。AI剪口播 把这步交给 agent:转录全文、标出该删的片段,再起一个**本地审核网页**,让你在波形上一眼看清「真正会切到哪一帧」(红=删、黄=算法补切、灰=静音),所见即所得。

它**不直接剪视频**,只把「哪里该删」想清楚,生成剪辑工程文件(FCPXML)交给专业软件做最后一步。无云端、无账号、无第三方图床 —— 音频 base64 直传火山引擎转录,剩下全在你本机跑。

### 主要特性

- **AI 标口误** — 重复重说、残句、句内卡顿、纯语气词整句,自动预选出来等你确认
- **波形审核网页** — 自绘 canvas 波形,长视频也顺滑;删除/补切/静音三色叠加,所见即所得
- **真 WYSIWYG** — 前后端共用同一份切割算法,预览到哪一帧,导出就是哪一帧,不漂移
- **剪映 / FCP 通吃** — 导出的 FCPXML 1.8 一个文件同时被剪映专业版和 Final Cut Pro 识别
- **自进化学习** — 说一句「学一下」,它从你这次的真实剪辑里抽规则,沉淀成你的个人偏好
- **100% 本机** — 视频和音频不经任何第三方存储,key 只在本地
- **跨平台自检** — 一条命令逐项告诉你缺什么、怎么补(Windows / macOS / Linux)

## 安装

### 让 coding agent 帮你装(最简单)

把这个仓库地址发给你的 agent(Claude Code、Codex、Kimi 等),说一句 **「装一下」**:

```text
https://github.com/lcbuaaliu/ai-jian-koubo
```

agent 会读 README 一步步带你装好,约 1 分钟。

### Claude Code 手动安装

把仓库 clone 下来,**重命名为 `AI剪口播`**,放进 Claude 的 skills 目录:

```bash
git clone https://github.com/lcbuaaliu/ai-jian-koubo.git ~/.claude/skills/AI剪口播
```

> ⚠️ 文件夹名必须是 `AI剪口播` —— skill 触发词和脚本路径都依赖这个名字。

然后配置火山引擎 API Key(放在 **skills 目录的上一级**,所有引擎共用这一个文件):

```bash
cp ~/.claude/skills/AI剪口播/.env.example ~/.claude/skills/.env
# 编辑 ~/.claude/skills/.env,把 your_api_key_here 换成真 key
```

去[火山引擎新版控制台](https://console.volcengine.com/speech/new/setting/apikeys)生成**一个** API Key,并开通两个资源(各 **20h 免费额度**,独立抵扣):**录音文件识别-极速版**(`auc_turbo`) + **标准版**(`auc`)。默认轮流用,吃满 ≈40h;只想用一个就在转录时加 `--flash` / `--v3-standard`。

最后跑一次自检,缺什么它会告诉你怎么补:

```bash
node ~/.claude/skills/AI剪口播/scripts/doctor.js
```

### 其它 Coding Agent(Codex / Gemini CLI / OpenCode 等)

核心流程不依赖 Claude Code。把这个仓库链接发给 agent,让它**从 [`SKILL.md`](SKILL.md) 开始读**,按里面写死的步骤 0-7 执行即可。它只需要在需要时加载引用到的脚本:

- `scripts/run_transcribe.sh` — 抽音频 + 火山引擎转录
- `scripts/gen_analysis.js` · `merge_selections.js` · `generate_review.js` — 分析与生成审核页
- `scripts/review_server.js` — 审核服务器 + FCPXML 导出
- `用户习惯/*.md` — 口误判断规则

有文件系统访问权限的 agent 可以帮你装到本地 skills 目录;没有的话,也能在当前会话里直接照 `SKILL.md` 执行一遍。

## 使用

装好后,在 agent 里直接说:

```text
帮我剪这个口播视频 /path/to/video.mp4
```

agent 会按 [`SKILL.md`](SKILL.md) 的流程跑:

1. 抽音频 → 火山引擎转录成字级字幕
2. 读全文,标出口误 / 口癖 / 残句,预选要删的片段
3. 起一个本地审核网页,波形三色标注真正会切到哪一帧
4. 你勾选确认 → 点「导出 FCPXML」
5. 把生成的 `*_cut.fcpxml` 拖进剪映 / Final Cut Pro,完成最终剪辑

它还有个**转字幕模式** —— 说「把这个视频转成字幕」,会输出规范的字幕文本(无时间戳的 markdown)。

## 工作原理

这个 skill 用**渐进式披露**:主入口 [`SKILL.md`](SKILL.md) 是一张工作流地图,其它文件按需加载。

| 文件 | 作用 | 何时加载 |
|------|------|----------|
| `SKILL.md` | 核心工作流与规则 | 始终(skill 触发时) |
| `scripts/run_transcribe.sh` | 抽音频 + 火山引擎 ASR | 步骤 1-4(转录) |
| `scripts/gen_analysis.js` | 生成口误分析的纯文本输入 | 步骤 5(分析) |
| `scripts/merge_selections.js` | 句号选择 → 词级 idx 展开 | 步骤 5(合并) |
| `scripts/generate_review.js` | 生成审核网页 | 步骤 6(审核) |
| `scripts/review_server.js` | 审核服务器 + `/api/fcpxml` 导出 | 步骤 7(审核) |
| `scripts/lib/compute_keeps.js` | 切割算法(前后端共用的唯一来源) | 预览与导出 |
| `scripts/doctor.js` | 跨平台环境自检 | 首次安装 |
| `用户习惯/*.md` | 口误判断规则 / 个人偏好 | 步骤 5(分析) |

**切割算法只有一份**(`compute_keeps.js`,UMD 模块):服务器生成 FCPXML 和前端波形预览复用同一份代码,保证「预览到哪一帧,导出就是哪一帧」。导出格式是 **FCPXML 1.8**,剪映专业版(文件 → 导入 → Final Cut Pro XML)和 Final Cut Pro 都能识别。

## 依赖要求

- 一个有文件系统访问、能跑 shell 命令的 coding agent
- `node` · `python3` · `ffmpeg` · `curl`(`doctor.js` 会按平台给安装命令)
- 一个[火山引擎](https://console.volcengine.com/speech/new/setting/apikeys)账号(用于语音转录,有免费额度)
- Claude Code **不是必需** —— 它只是众多可用 agent 之一

## 隐私

`.env`(你的 key)被 `.gitignore` 忽略,不会进仓库。音频默认走 base64 直传火山引擎,不经任何第三方图床。

## 致谢

由 **栗氪聊AI** 创建。

## License

[MIT](LICENSE) — 随便用、改、分发。
