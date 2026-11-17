---
name: 'pms-workflow'
description: 'PMS capability skill，负责 PMS 机械读写、状态流转、回填和附件上传，包括创建业务需求、将 ITR 以 solution=5 正式关联到新需求，以及上传本地 Wiki / 文档。'
---

# PMS Workflow

使用这个技能处理聚焦的 PMS 任务。

它负责：

1. 读取 PMS bug / 需求 / 线上反馈
2. 按 ID 拉取 bug / 线上反馈详情，并补齐评论、附件、操作记录、图片与链接
3. 做 PMS 状态流转、评论回填、提测
4. 在团队任务下创建子任务，并按指定代码仓库 / DevOps 项目精确关联
5. 处理 PMS session / storageState 探测
6. 使用 PMS token 登录接口换取 `JSESSIONID`，并写入 Playwright storageState
7. 打开 PMS 浏览器登录后，必须通过 `/pdca/user/token/current` 捕获并落盘 `pms_utk_...`，后续用它刷新 `JSESSIONID` 并写入 `pms.json`
8. 通过统一 `pms-workflow.cjs work-item` 预览并创建产品需求、Bug，完成 ITR `转业务需求`，给已有产品需求 / Bug 上传验收资产，或给已有业务需求上传原始文档并回读确认
9. 审计 PMS 版本的真实关联 Bug、状态分布和正式验收包覆盖率
10. 使用本地批次门禁审计治理结果、5 轮预算、独立上传清单、正式 summary 和上传回读

## 当前阶段

- 这是新的 PMS capability 入口
- Design 端到端任务优先由 `design-workflow` 调用本技能

## 默认原则

- 优先先用脚本，不要默认手动筛选 PMS 页面
- Wiki 指引用于解释对象、角色、流程和状态的业务语义；当前 Swagger、接口响应和写后回读才是机械执行事实源。两者冲突时保留差异，不用旧页面说明覆盖当前接口事实
- PMS 工作台、默认列表和“我的”视角通常只包含当前用户相关、未完结或未归档数据，不能直接作为全量统计。全量审计必须显式取消个人 scope、完整分页或使用版本专用关联接口
- 所有 PMS 流程优先走 PMS 接口并使用 storageState 里的 `JSESSIONID` 作为凭证；接口覆盖不了时再走 Playwright 页面流程；只有定位页面结构、登录态异常或交互脚本失效时，才做手工网页调试
- 如果当前 PMS 流程在 `references/pms-api-inventory.md` 里没有记录，先查 PMS 接口文档 `https://pms.3weijia.com/pdca/doc.html#/home` 或 Swagger JSON `https://pms.3weijia.com/pdca/v2/api-docs`，确认接口后再实现脚本
- 默认无头运行；只有登录、会话排障或必须核对页面细节时，才补 `--headed`
- 用户已经给了详情链接时，优先直接打开详情链接本身
- 读取 bug、ITR、需求时，不只读文字；遇到截图、图片、录屏时，把视觉内容也纳入理解
- 读取 ITR / 线上反馈详情时，优先用 `scripts/pms-workflow.cjs feedback detail --feedback-id <id>`，不要只依赖页面可见文本
- 查询“我的 Bug / 线上反馈 / 产品需求”统一使用 `scripts/pms-workflow.cjs items list --type <bug|online-feedback|requirement>`，不要再为每种工单维护一套分页、scope 和表格输出脚本
- 产品线列表可以先整体拉取，但在复用同一份登录态时，多条详情建议串行读取
- 已有 `pms_utk_...` 这类 PMS 登录 token 时，优先用 `scripts/pms-workflow.cjs auth login token` 换取 `JSESSIONID` 并写入 cookie，不要要求用户手动登录
- `pms.json` 或 storageState 文件存在不代表登录态有效；只有 `/pdca/bug/list` 返回 HTTP 200，且 JSON `code=0`、`success=true`，才视为 PMS 凭证可用
- PMS 登录文件最小集是用户级长期 token + 项目级 storageState：正式环境长期 token 默认在用户级凭证目录 `pms-user-token.json`，测试环境是 `design-pms-test-user-token.json`；当前业务项目 `.tmp/pms.json` / `.tmp/design-pms-test.json` 只是用 token 换出的 `JSESSIONID` storageState 缓存
- 用户级凭证目录按 `playwright-workflow` 的 `DEFAULT_LONG_LIVED_CREDENTIAL_DIR` 解析：Windows 默认 `%LOCALAPPDATA%\playwright-storage-state`，Linux/macOS 默认 `${XDG_STATE_HOME:-~/.local/state}/playwright-storage-state`。不要递归扫描业务仓库找 `pms-user-token.json`
- `pms_utk_...` 是 PMS 长期凭证，必须落盘到用户级 token 文件；`JSESSIONID` 是用它换出来的当前 cookie，写入当前项目 `.tmp` 下的 storageState
- PMS 脚本默认正式环境：不传 `--env` 时就是 `https://pms.3weijia.com`、用户级 `pms-user-token.json` 和当前项目 `.tmp/pms.json`。`pmstest` 只用于显式 `--env test` 或 `--base-url https://pmstest.3weijia.com` 的测试场景
- PMS 页面跳到 `login.3weijia.com`、接口返回 `302 -> /pdca/login`、返回 HTML 登录页，或只有 cookie 但接口验证失败时，先刷新 storageState，不要继续关单、提测或分析登录页
- 浏览器登录必须发生在 `scripts/pms-workflow.cjs auth login browser` 启动的浏览器上下文里；用户在其他浏览器窗口登录不会自动同步到该脚本
- PMS 的 `JSESSIONID` 可能写在 `/pdca` 路径下；验证 cookie 时优先访问 `/pdca/bug/list`，不要只用站点根路径判断
- 产品需求、Bug 及其复现 / 验收附件写入统一使用 `pms-workflow.cjs work-item`；默认只做本地校验和预览，只有用户确认后显式传 `--apply` 才允许远程写入
- PMS 的 `问题原因` / `详细解决方案` / 评论先写项目、产品和用户能直接理解的业务结论，再用独立“技术说明”段落保留文件、API、runtime、Gerrit 和日志细节。不得用文件路径、类名、接口或内核状态直接开头，也不得只写“已修复”而不说用户问题如何变化
- PMS 正文不得包含只有创建者机器可访问的路径，包括 `/disk2/...`、`D:\...`、`C:\...`、`~/...` 和 `wiki/...`。正文依赖本地 Wiki 或文档时，必须上传 PMS 正式附件并回读文件名
- 对象类型必须严格区分：业务需求使用 `userRequirement`，产品需求使用 `productRequirement`，Bug 使用 `bug`；不得用产品需求的 `objType` 给业务需求绑附件
- 创建前按精确标题查重，默认阻止重复工单；只有用户确认确实需要同名记录时才传 `--allow-duplicate`
- 已有 Bug 开始处理时，先真实复现，再用 `pms-workflow.cjs bug confirm` 选择缺陷确认方案并回读。不得在未复现时抢先认领，也不得等到 Gerrit 提交或验收后才首次改状态。`bug confirm` 默认 dry-run，不调用 `submitTest`。
- 缺陷确认必须支持当前 PMS 的 `代码修复 / 标准功能 / 设计缺陷 / 无法重现 / 重复BUG / 配置导致 / 外部原因` 分流。只有 `代码修复` 进入源码修改；`配置导致` 保留待处理责任但不进入代码链路，其他方案按证据完成非代码收口。
- `bug resolve` 真正流转到已解决时，`解决人` 默认使用当前 PMS 登录操作人，不沿用历史空值或旧解决人。只有业务确实需要代为解决时才显式传 `--resolve-by-user` / `--resolve-by-name`；写入后必须同时回读 ID 和姓名
- `bug resolve` 默认 dry-run。正式写入必须先用 `--output-json <preview.json>` 保存预览，再传 `--apply --preview-file <preview.json>`；CLI 会在任何写入前比较当前 Bug 与计划 payload，发现状态或参数漂移立即中止。
- PMS 解决流程按 `editBug → edit readback → submitTest → status readback → solution owner edit → final readback` 串行执行。任一步接口成功但回读不一致时输出 `PMS_PARTIAL_WRITE`、已完成步骤和实际状态，并停止后续写入；不得盲目整套重跑。
- 正式验收回填只接受通过本地 `design-workflow.cjs 3d acceptance-gate ... --pms-writeback` 的 `PASS/FAIL/BLOCKED`。`INCONCLUSIVE_SCRIPT` 与 `blockedCategory=script` 只留在本地，不上传 acceptance ZIP、不写 PMS 验收结论。

## 最小流程

1. 先确认当前任务是 Bug、产品需求、业务需求、任务、迭代、版本，还是 ITR / 线上反馈，并识别所处业务阶段。
2. 先用 `scripts/pms-workflow.cjs auth probe --require-valid` 验证当前 storageState 是否已经能通过 PMS API。
3. 优先用列表或详情脚本读取结构化结果，再按需要补页面字段；默认列表只能解释当前筛选，不能宣称全量。
4. 已有 Bug 先完成复现后缺陷确认；只有代码修复分支进入开发。其他任务根据类型执行评论回填、状态流转、提测、已解决回填、ITR 处理或团队任务子任务创建。
5. 写入前 dry-run，写入后回读目标对象、状态、关联和附件；钉钉通知或页面提示不能替代回读。
6. 如果任务还需要联动 Gerrit 或跨系统收口，交回 `design-workflow` 继续编排。

PMS 解决文案的默认结构：

```text
业务结论 / 处理结论：用一到两句说清这是什么问题、现在如何处理、什么时候生效。

用户说明 / 后续安排：说明用户是否需要重试、重建数据、升级版本或跟进需求。没有明确证据时不猜测。

技术说明：放文件、模块、根因、Gerrit、构建版本和验收证据，供研发追溯。
```

## 产品需求、Bug 与验收附件写入

统一入口：

```powershell
node scripts/pms-workflow.cjs work-item requirement --draft-file .tmp\<task>\requirement.json
node scripts/pms-workflow.cjs work-item requirement-artifact --requirement-no <requirementNo> --artifact-kind acceptance --attachment .tmp\<task>\acceptance.cjs --attachment .tmp\<task>\acceptance.config.json --attachment .tmp\<task>\<timestamp>\acceptance.webm --attachment .tmp\<task>\<timestamp>\07-final-state.png --attachment .tmp\<task>\<timestamp>\summary.json --attachment .tmp\<task>\<timestamp>\execution-context.json --attachment .tmp\<task>\<timestamp>\console.json --attachment .tmp\<task>\<timestamp>\network.json
node scripts/pms-workflow.cjs feedback handle --feedback-id <itrId> --solution-name 转业务需求 --requirement-draft-file .tmp\<task>\user-requirement.json --attachment <local-document> --dry-run
node scripts/pms-workflow.cjs work-item itr-user-requirement --itr-id <itrId> --draft-file .tmp\<task>\user-requirement.json --attachment <local-document>
node scripts/pms-workflow.cjs work-item user-requirement-document --requirement-id <userRequirementId> --attachment <local-document> --remove-local-path <path-in-description> --itr-id <itrId>
node scripts/pms-workflow.cjs work-item bug --draft-file .tmp\<task>\bug.json --artifact-kind repro --attachment .tmp\<task>\repro.cjs --attachment .tmp\<task>\acceptance.config.json --attachment .tmp\<task>\<timestamp>\repro.webm --attachment .tmp\<task>\<timestamp>\execution-context.json
node scripts/pms-workflow.cjs work-item bug-artifact --bug-id <bugId> --artifact-kind acceptance --attachment .tmp\<task>\acceptance.cjs --attachment .tmp\<task>\acceptance.config.json --attachment .tmp\<task>\<timestamp>\acceptance.webm --attachment .tmp\<task>\<timestamp>\07-final-state.png --attachment .tmp\<task>\<timestamp>\summary.json --attachment .tmp\<task>\<timestamp>\execution-context.json --attachment .tmp\<task>\<timestamp>\console.json --attachment .tmp\<task>\<timestamp>\network.json
```

以上命令默认 dry-run，不产生远程写入。用户确认后才追加 `--apply`。脚本会完成查重、创建或上传、评论和写后回读；任一步失败都会返回非零退出码。

`requirement`、`requirement-artifact`、`bug`、`bug-artifact` 的 `--attachment` 继续按原有契约统一打成 ZIP，`repro` / `acceptance` 的 WebM、summary、截图和日志要求不变。`user-requirement-document` 仅用于已有业务需求的普通说明文档，保留 `.md` / PDF / Office 等原文件名和类型直接上传，不代替验收包 ZIP。它会先回读最新对象，仅删除显式传入的 `--remove-local-path`；路径删除后只剩“详细调查清单”等纯引用标签时删除整个段落，仍有业务文字时保留段落。编辑 payload 仅包含 Swagger `UserRequirementVo` 白名单字段，保留所有可编辑值并排除 GET 额外元数据。之后验证 `description` 无本机路径、`objType=userRequirement` 附件名已回读；传 `--itr-id` 时还必须回读到 `solution=5` 且 `objId` 等于目标业务需求 ID。

ITR 真正执行 `转业务需求` 使用 `pms-workflow.cjs feedback handle --solution-name 转业务需求`，内部委托 `work-item itr-user-requirement` 模式。该流程先回读 ITR，按页面真实规则用 ITR 标题、正文、创建人和产品线补默认值，合并草稿中的指派人、类型、期望上线等必填字段，调用 `/userRequirement/save` 创建业务需求，再调用 `/onlineFeedback/onlineQuestionConfirm` 写入 `solution=5` 和新需求 `objId`并回读 ITR，最后上传并回读原始文档。如附件阶段失败，必须使用已返回的需求 ID 进入 `work-item user-requirement-document` 修复，不重复创建。

大附件上传不得把 ZIP 内容转成 Base64 嵌入 Playwright `run-code`。当前 CLI 只把受限的本地 ZIP 路径交给子进程，由 Node 侧使用文件型 Blob 和当前 PMS cookie 上传；ZIP 生成、上传和子进程回读都不复制整包附件字节。Windows 出现 `spawnSync ENOBUFS` 时，先确认实际调用的是当前技能里的 `pms-workflow.cjs work-item`；禁止回退一次性旧脚本、拆散 ZIP 或跳过附件回读。

## 版本验收审计

版本完整 Bug 总体以 `GET /pdca/version/listBugV2?versionId=<id>` 为准；`POST /pdca/bug/list` 的 `resolveVersionId` 查询只作差异对照，不能用于宣称版本总数。使用只读脚本统一统计状态和正式验收包：

```bash
node scripts/pms-workflow.cjs version audit --version-id <versionId> \
  --output-json .tmp/pms-version-<versionId>/audit.json \
  --output-markdown .tmp/pms-version-<versionId>/report.md
```

正式 Bug 验收包必须匹配 `bug-<bugId>-acceptance-<yyyyMMdd-HHmmss-UTC8>.zip`。附件查询失败时结果必须标记为不完整，不能把查询失败写成缺包。完整口径见 `references/pms-version-acceptance.md`。

提供治理基线时追加 `--baseline-json <snapshot.json>`，输出会明确区分原始基线、当前 `listBugV2` 关联总体、关联漂移和 resolveVersion 对照。批次上传前后使用纯本地门禁：

```bash
node scripts/pms-workflow.cjs acceptance validate-batch --results-json <merged-results.json> --manifest <upload-manifest.json> --version-audit <audit.json> --apply-log-root <task-root>
```

正式新 acceptance 的 `summary.json` 必须为 PASS，包含非空且全部通过的 `requiredAssertions` / `targetAssertions`；`risks` 仅披露旁路风险。旧 `assertions` 只有显式传 `--allow-legacy-acceptance-summary` 才兼容。

## Token 登录

当用户提供 PMS 登录 token，或明确说“用 token 获取 cookie / JSESSIONID”时，优先使用：

```powershell
$env:PMS_LOGIN_TOKEN = '<pms_utk_token>'
node scripts/pms-workflow.cjs auth login token
```

也可以直接传 `--token <pms_utk_token>`；如果在交互终端运行 `token` 模式且没有传 token，脚本会提示用户粘贴 token。该模式会覆盖写入目标 storageState，相当于替换当前 PMS cookie 配置。

脚本会调用 `POST /pdca/token/login`，把返回的 `JSESSIONID` 写入对应 PMS 域名的 cookie，并保存为当前项目 Playwright storageState。默认正式环境保存到 `.tmp/pms.json`；只有显式 `--env test` 时才保存到 `.tmp/design-pms-test.json`。如果没有显式传 token，脚本优先读取用户级默认 token 文件：正式环境 `%LOCALAPPDATA%\playwright-storage-state\pms-user-token.json`，测试环境 `%LOCALAPPDATA%\playwright-storage-state\design-pms-test-user-token.json`；Linux/macOS 使用对应的 `${XDG_STATE_HOME:-~/.local/state}/playwright-storage-state`。

当用户没有现成 token，但可以在浏览器登录 PMS 时，使用：

```powershell
node scripts/pms-workflow.cjs auth login browser --env prod
```

脚本会打开可见浏览器，用户必须在这个脚本启动的浏览器里完成登录。登录完成后脚本必须在浏览器上下文里调用 `GET /pdca/user/token/current`，确认返回 JSON 且包含 `pms_utk_...`，把完整 token 写入默认 token 文件，再调用 `POST /pdca/token/login` 换取并写入最终 cookie。不能只因为页面出现 `JSESSIONID` cookie 就判断登录成功；统一登录页也可能产生无效 cookie。不再依赖点击页面里的“用户Token”菜单。

如果用保存的 `pms_utk_...` 换 `JSESSIONID` 失败，脚本允许 fallback 一次：打开浏览器重新捕获新的 `pms_utk_...`，覆盖用户级 token 文件，再重试 token 登录。完整 token 和 `JSESSIONID` 都是敏感凭证，只能写入本机凭证目录或当前项目 `.tmp`，不要写进日志、回复或长期记忆。

如果机器没有系统 Chrome，优先传 `--browser chromium` 或设置 `PLAYWRIGHT_EXECUTABLE_PATH=<chrome|chromium路径>`；不要因为系统 Chrome 缺失就放弃 PMS token 登录。

token、`JSESSIONID` 都是敏感凭证，日志和回复里只能展示脱敏值或文件路径。

## 团队任务子任务创建

当用户要求“在主任务下建子任务”“按当前修改仓库建子任务”“关联仓库”时，优先使用：

```powershell
node scripts/pms-workflow.cjs task create-subtasks `
  --parent-task-no <task-no> `
  --repo <repo1,repo2> `
  --project-name <devops-project-name> `
  --dry-run
```

确认 `selectedLibraries / payloads` 里的 `projectName`、`projectId`、`libraryUrl` 正确后，再去掉 `--dry-run` 创建。若同名仓库在多个项目分支中存在，必须传 `--project-name` 或 `--project-id`，不要只按仓库名创建。

## 线上反馈 / ITR 详情读取

当用户给出线上反馈、ITR ID 或详情链接，并需要判断问题背景、客户信息、截图、沟通记录或后续联动 `dbg` / Gerrit 时，优先运行：

```powershell
node scripts/pms-workflow.cjs feedback detail --feedback-id <id> --output-json C:\temp\itr-<id>.json
```

脚本会输出并保存结构化结果：

- 基础字段：标题、状态、问题类型、产品线、严重程度、轮值团队、指派人、用户账号、方案号、版本号、是否 3D、是否需要远程、远程码
- 富文本与页面信息：正文富文本块、页面文本、表单控件回读值
- 关联数据：评论、附件、操作记录
- 视觉与链接线索：从正文、页面富文本块、评论和附件里汇总 `imageUrls`，从正文和评论里汇总 `linkUrls`
- 图片来源拆分：`imageSources.contentImageUrls / commentImageUrls / attachmentImageUrls` 可用于确认图片来自正文、评论还是附件

读取结果用于排查时：

- 后续需要查询方案或历史时，优先复用 `schemeId / userAccount / version`
- 只要 `imageUrls` 非空，就把图片内容纳入问题理解，不能只看 `pageText` 或评论文本
- 多个 ITR 详情建议串行读取，避免复用同一份 PMS 登录态时相互影响

## 先读哪些文档

- PMS 业务操作模型：
  [./references/pms-operating-model.md](./references/pms-operating-model.md)
- PMS 检查清单：
  [./references/pms-checklist.md](./references/pms-checklist.md)
- PMS 脚本速查：
  [./references/pms-script-recipes.md](./references/pms-script-recipes.md)
- PMS 接口清单：
  [./references/pms-api-inventory.md](./references/pms-api-inventory.md)
- PMS 版本 Bug 与验收包审计：
  [./references/pms-version-acceptance.md](./references/pms-version-acceptance.md)
- 需求单补评论并提测：
  [./references/pms-product-requirement-submit-test.md](./references/pms-product-requirement-submit-test.md)
- 浏览器 / 会话经验：安装 `playwright-workflow` 后读取该技能的 `references/browser-session-notes.md`

## 依赖边界

- 本技能依赖：`playwright-workflow`
- 单独安装本技能时，应通过 SkillHub 同时安装 `@global/playwright-workflow`，并保持两个技能位于同一 `skills` 目录；也可改装 `@global/design-dev-workflow` 聚合包。
- 仅浏览器登录或首次初始化共享 Playwright runtime 时需要该依赖。缺少时 CLI 会给出安装建议，但不会自动下载；不需要浏览器的 PMS 命令保持可独立运行。

## 组合方式

- 面向聚焦的 PMS 查询、状态流转与回填任务
- 由 `playwright-workflow` 提供浏览器 runtime、session 与 storageState 基线
- 在跨系统闭环场景中，由 `design-workflow` 组合调用