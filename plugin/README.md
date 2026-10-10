# code-edit-trace

课堂代码轨迹采集插件。学生打开作业文件夹后，插件会立即要求填写姓名、班级、学号；完成填写前不会开始记录。随后插件把代码差异和文件操作追加写入工作区根目录的 `edit-trace.log`。0.0.6 起还会自动采集与当前工作区匹配的 Cline AI 会话，写入 `ai-session-trace.log`。

## 插件行为

- 保存与外部写入都会记录：学生手动保存时按整份文件计算一次有效差异；Cline 等插件直接写文件时，通过文件系统变化记录 `origin:"disk"` 的编辑。
- 文件创建去重：`file.create` 只写一条，并尽量在事件后读取文件首内容，不再固定输出空 `changes`。
- 身份门禁：每个新工作区文件夹在打开时填写一次，后续打开同一文件夹不会重复填写；关闭填写窗口不会跳过，插件会立即再次弹出，直到信息填写完整。
- 启动时序：文件监听会在后台校时和身份填写之前注册，学生在打开文件夹后立即让 Cline 创建或修改文件也会进入采集流程。
- 文件操作记录：记录工作区内的文件/文件夹新建、删除、重命名、移动，以及 Cline/外部工具直接产生的文件写入。
- 日志格式：每行一条 UTF-8 JSON，包含学生信息、事件类型、文件路径、差异内容、字符/行数变化和是否大段写入。
- 大段写入不再直接判定为 `suspicious`：新日志会写 `bulkWrite` 用于统计大段代码，`suspicious` 保留为兼容字段但默认 `false`。
- AI 会话采集：自动扫描 `%USERPROFILE%\.cline\data\sessions`，只采集 `workspace_root` 与当前工作区匹配的 Cline 会话；记录提问、AI 回答、工具调用摘要、模型和 token。
- AI 日志压缩：`ai-session-trace.log` 每个学生与 `sessionId` 只保留一条最新完整记录，旧版重复快照会在启动时自动合并。
- 系统消息过滤：过滤 Cline 注入的 `mode_notice`、`file_content`、`environment_details` 等系统内容，不计入提问或首次提问时间。
- 可信时间：通过 HTTPS `Date` 响应头校正学生机时钟，日志同时保留原始时间和校正时间。
- 防递归：插件不会记录 `edit-trace.log` 和 `ai-session-trace.log` 自身。

## 终端执行步骤

在 `code-edit-trace` 目录下执行：

```powershell
cd "D:\Desktop\AI辅助编程-课程资源开发\code-edit-trace"
npm install
npx tsc -p .
node -e "global.File=require('node:buffer').File; require('@vscode/vsce/out/main')(['node','vsce','package','--out','code-edit-trace-0.0.7.vsix','--allow-star-activation','--allow-missing-repository','--skip-license'])"
```

最终产出：

- `out/extension.js`
- `code-edit-trace-0.0.7.vsix`

如果安装 Node 20/22 LTS，也可以使用简化命令：

```powershell
npx vsce package --out code-edit-trace-0.0.7.vsix --allow-star-activation --allow-missing-repository --skip-license
```

## 学生端操作说明

1. 安装：打开 VS Code 扩展面板，从 VSIX 安装 `code-edit-trace-0.0.7.vsix`。
2. 使用：打开作业文件夹，按弹窗提示填写姓名、班级、学号。未完整填写前，插件会持续要求填写，不会记录代码改动或 Cline 会话。
3. 之后正常修改、保存、新建/删除/重命名/移动文件即可。
4. AI 会话会在后台自动采集；如果漏采，可在命令面板执行 `Code Edit Trace: 手动采集 AI 会话数据`。
5. 上交：源代码文件夹 + `edit-trace.log` + `ai-session-trace.log`。

如果填错身份，学生可在 VS Code 命令面板中执行：

```text
Code Edit Trace: 填写学生信息
```

## 0.0.7 修复

- 修复打开文件夹后立即使用 Cline 时，网络校时可能阻塞文件监听注册、导致第一批操作漏记的问题。
- 身份填写改为模态门禁；关闭弹窗或 Esc 取消任一输入框后会自动重新提示，未填写完整前不开始采集。
- 统一编辑快照和磁盘快照的文件键，避免返回原有文件后，下一次外部写入重复包含上一次保存内容。

## 教师分析

```powershell
cd "D:\Desktop\AI辅助编程-课程资源开发\code-edit-trace"
$env:PYTHONIOENCODING = "utf-8"
python analyse_trace.py "学生上交路径\edit-trace.log"
```

脚本会：

- 兼容新旧日志格式；
- 跳过损坏 JSON 行；
- 输出每位学生的编辑次数、文件数、文件操作数、插入/删除字符数；
- 统计单次改动超过 300 个字符的大段写入，但不把它当成“可疑作弊”；
- 输出文件新建、删除、重命名、移动记录。

## 整体课堂架构说明

课堂完整工具栈：普通桌面 VS Code。

- CodeTour 插件：制作分步编程任务，下发作业模板。
- Live Share 插件：教师课堂演示，对个别学生远程协助调试。
- `code-edit-trace.vsix`：采集代码保存差异、Cline/外部写入、学生身份、文件操作和 AI 会话记录。
- `analyse_trace.py`：教师批量分析日志，统计编辑量和文件操作，识别大段代码写入。

## AI 会话自动采集

0.0.6 起插件会自动扫描本机 Cline 会话目录，默认每 30 秒检查一次。插件只处理 `workspace_root` 与当前打开工作区一致的会话，并把数据写入工作区根目录的 `ai-session-trace.log`。

```text
%USERPROFILE%\.cline\data\sessions\<session-id>\<session-id>.json
%USERPROFILE%\.cline\data\sessions\<session-id>\<session-id>.messages.json
```

`<session-id>.json` 里有学生最初提示词、模型、token、成本和工作区路径；`.messages.json` 里有用户提问、AI 回复、`editor`/`run_commands`/`read_files` 等工具调用及执行结果。

日志每条记录包含：

- 会话 ID、标题、初始提示词、模型、provider、工作区路径
- 学生提问、AI 文本回答、工具调用名称和参数摘要
- 工具结果预览、token 和成本

插件不会读取 `providers.json` 和 `secrets.json`，不会导出 API Key。`thinking` 内容不会写入日志。

插件通过多个 HTTPS `Date` 响应头计算本机时钟偏移。联网校时失败时继续记录本地时间，并写入 `timeTrusted:false`，教师面板会显示时间未校准提示。

相关设置：

- `code-edit-trace.enableAiSessionCapture`：是否自动采集 AI 会话，默认 `true`。
- `code-edit-trace.enableTimeSync`：是否启用网络校时，默认 `true`。
- `code-edit-trace.timeSyncUrls`：HTTPS 校时地址列表，可替换为学校内网时间服务。
- `code-edit-trace.timeSyncIntervalMinutes`：重新校时间隔，默认 30 分钟。

当前仓库也保留了 `collect_cline_sessions.py` 作为离线兜底脚本，可直接把本机或上交的 Cline 会话目录导出成 JSONL：

```powershell
cd "D:\Desktop\AI辅助编程-课程资源开发\code-edit-trace"
$env:PYTHONIOENCODING = "utf-8"
python collect_cline_sessions.py "$env:USERPROFILE\.cline\data\sessions" "ai-session-trace.jsonl"
```

该脚本输出为独立 JSONL，适合没有安装 `0.0.6` 插件的环境；正常课堂采集建议使用插件自动生成的 `ai-session-trace.log`。

## 已知限制

- 外部文件写入只在 VS Code 保持打开且工作区仍打开时记录。
- 外部重命名/移动采用短时间“删除 + 新建”配对推断，可能把“复制后立即删除原文件”误判为移动。
- 插件记录的是文件落盘变化；AI 会话记录依赖 Cline 汉化版在本机生成 `<session-id>.json` 和 `.messages.json`。
- 如果学生机使用共享 Windows 账号，`.cline` 目录会被多人共享，不同学生的 AI 会话可能混在一起。
- 网络校时需要访问至少一个配置的 HTTPS 地址；完全离线时日志仍会生成，但 `timeTrusted` 为 `false`。
