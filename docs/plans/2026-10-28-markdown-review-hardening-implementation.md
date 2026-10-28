# Markdown 批注功能:技术修复实施方案

日期:2026-10-28。输入:《2026-10-27-markdown-review-hardening-plan.md》(v4,决策已全部落定)。本文把每项决策展开到**文件级、函数级**的改动说明,按 PR-1…PR-5 组织,每项含改动点、测试与验收。存放规则同计划文档:仅存在于 `docs/markdown-review-hardening-plan` 分支,不进入 PR 源分支。

术语对照:计划文档的"问题 N"在下文保持同一编号;**DoD** = 该 PR 的完成定义。

---

## 0. 总览

| PR | 内容 | 规模 | 依赖 |
| --- | --- | --- | --- |
| PR-1 | 问题 1(1a 门禁 / 1b 影响判断 / 1c 语言缓存 / 1d 条件广播)+ 问题 12 常量提取 + deprecated 顺手修 | M | 无 |
| PR-2 | 问题 2(孤儿三层防御)+ 问题 10(提交竞态) | M | 无(与 PR-1 可并行,建议先后) |
| PR-3 | 问题 3(去除 persist 翻转)+ 问题 9(草稿撤回) | S | PR-2(回执协议) |
| PR-4 | 问题 5(interim+幂等)、6(立即回错)、7(错误编号化)、11(8 语言补齐) | M–L | 问题 7 依赖问题 6 的回包通道 |
| PR-5 | 233 校验目标提交、verifier 流程、技术债登记、PR 瘦身、问题 12 全库复查 | S | 全部之后 |

所有前后端协议改动与插件**同包发布**(buildPlugin 同时打包 webview 产物),无跨版本兼容负担。

---

## 1. PR-1:性能与零开销

### 1.1 问题 1a:创建层门禁 + 开关文案

**改动点**

1. `NativeMarkdownExtensionProvider.createBrowserExtension`(NativeMarkdownExtensionProvider.java:12):
   - 在现有基线/参数检查后,新增 `MarkdownReviewService.getInstance(panel.getProject()).getConfig().enabled` 检查;`false` → 返回 `null`(默认路径零开销,不再构造 `NativeMarkdownAnnotations`)。
2. `NativeMarkdownAnnotations`(已存在的实例,负责"关闭即时生效"):
   - 新增字段 `boolean lastPublishedEnabled`;`publish()`(:368)开头先查 `service.getConfig().enabled`:
     - `false` → 组装**精简状态**(仅 `kind/panelId/enabled/documentVersion/dirty/language/submitShortcut`,不含 `annotations`、`diagrams`)发送一次;若 `lastPublishedEnabled` 已为 `false` 则**直接返回**(不重复发)。发布前置 `lastPublishedEnabled = false`。
     - `true` → 走现有全量路径,置 `lastPublishedEnabled = true`。
   - 文档监听器(构造器 :79-81 与 `dispose` :399-401)改为随 `enabled` 增删:新增 `syncDocumentListener(boolean enabled)`;在每次 `publish()` 的开关判断处调用(开关变化由 service listener 的 `changed()` 推过来,即时生效)。
   - **deprecated 顺手修**:注册改用带生命周期归属的重载 `document.addDocumentListener(listener, service)`(`service` 实现 `Disposable`,项目关闭自动摘除;`dispose()` 里的手动移除保留)。
3. 设置开关文案:`webview/src/i18n/locales/*.json` 的 `markdownReview.settings.enabled.hint` 追加"开启后,已打开的预览需关闭重开后才会出现批注浮层"(10 语言文案在 PR-4 的 11 号问题里统一补齐,本 PR 先落 en/zh,其余语言暂回退英文——与计划文档 11 号决策一致,PR-4 收口)。

**测试**:provider 门禁(把"是否启用"判断抽成包内静态方法 `static boolean shouldCreate(Project)`,单测 enabled=false/true 两态);publish 精简短路(state 级:构造 service+panel mock 困难,改为把"组装精简状态"抽成纯函数断言字段集合)。
**验收(DoD)**:冒烟场景新增断言——enabled=false 时预览打开、连续按键,`BrowserPipe` 出站消息数为 1(仅首帧精简状态)或 0(若首帧前已禁用);verifier 报告 deprecated 列表中 `NativeMarkdownAnnotations` 的 `addDocumentListener` 项消失。

### 1.2 问题 1b:影响判断为主,时间窗只作合并

**核心设计(按用户拍板方向)**:后端**不再**在每次文档变更时主动推送;推送由三种事件驱动——前端 `sync`/`ready` 请求、用户操作(create/update/delete/complete/transfer)、设置变化。文档变更只做"影响分级标记"。

**改动点**

1. `NativeMarkdownAnnotations.documentListener`(:63-69):`documentChanged` 只置 `dirty = true` 并把事件区间登记给 service,**删除直接 `publish()`**。
2. `MarkdownReviewService.track()` 的 DocumentListener(:372-430):`beforeDocumentChange` 已计算 `affected` 集合;`documentChanged` 末尾(:428)改为按影响分级通知:
   - 新增包内枚举 `EditImpact { NONE, SHIFT, HIT }` 与纯函数 `static EditImpact classify(DocumentEvent event, Collection<Annotation> annotations)`:
     - 编辑区间在所有批注之后(且无 autoDeleteAffected 命中)→ `NONE`;
     - 编辑点在批注之前、未命中任何批注区间 → `SHIFT`(位置平移,marker 由平台自动更新,`item.start/end/startLine/endLine` 的同步照旧);
     - 命中批注区间(`affected` 非空)→ `HIT`。
   - `NONE` → 不调 `changed()`;`SHIFT` → 调 `changed(LIGHT)`;`HIT` → 执行现有受影响清理/状态维护后调 `changed(FULL)`。
   - `changed(...)` 增加重载:`changed()` 保持全量语义供操作类路径;`changed(LIGHT)` 通知面板做**轻量发布**。
3. `NativeMarkdownAnnotations.publish(boolean light)`:
   - `light=true`:不发 `diagrams`(`mermaidRanges` 全文扫描跳过——dirty 期间前端本来禁止创建批注,Mermaid 校验用不到);`annotations` 照发(位置已是 marker 更新后的值);语言取 1c 缓存。`dirty` 置 `true` 依旧。
   - `light=false`:现有全量路径。
4. 前端驱动的合并节拍:前端在 IDE 渲染补丁后回发 `sync`(`afterPatchListeners`),后端 `handle("ready"/"sync")` 的现有 `reconcile + publish` 即一次编辑后的完整收尾——**时间窗不需要后端自建**;`Alarm` 合并器列为后备(若实测 sync 频率高于每秒数轮再加,合并的是 sync 的应答)。
5. 前端 `index.ts`:无需改动(收到 dirty 状态即展示"等待同步",行为与现状一致;dirty 的获知时机从"按键瞬间"变为"下一次 sync/操作",间隔为 IDE 自己的渲染节拍,可接受)。

**测试**:`classify` 纯函数单测(三档各若干用例:编辑点在批注前/后/区间内/区间边界);service 级:注册计数 listener,`NONE` 编辑后通知数为 0、`HIT` 后为 1(用 `EditorFactory.createDocument` + WriteAction 驱动)。
**DoD**:打字期间(有批注的文件)后端出站消息 ≤ 每秒个位数;`NONE` 场景按键零消息;冒烟"受影响清理、未修改原文保留"既有用例不回归。

### 1.3 问题 1c:语言缓存(显式失效)

**改动点**

1. `LanguageConfigService`(:78-97):新增 `private static volatile String cachedUserLanguage`;`getUserLanguage` 命中缓存直接返回,未命中才调 `settingsService.getUserLanguage()`(磁盘读)。`getCurrentLanguage`/`getLanguageConfig` 自动受益。
2. 失效入口:新增 `public static void invalidateUserLanguageCache()`;在 `SettingsHandler.handleSetUserLanguage` / `handleClearUserLanguage` 成功路径调用(两处已各自调用 `refreshMarkdownPreviewLanguage()` → `refreshPreviewSettings()`,在同一位置加一行失效即可,SettingsHandler.java:409-413 附近)。
3. 日志:删除 :91 的逐次 `LOG.info`(改为 `LOG.debug`);首读后打一条 `LOG.info`(一次性)。

**测试**:注入假 `CodemossSettingsService`(计数 `getUserLanguage` 调用次数):连续两次 `getCurrentLanguage` 只触发一次磁盘读;`invalidate` 后再次触发;`setUserLanguage` 路径调用失效后,下一次 `getCurrentLanguage` 返回新值。
**DoD**:按键路径不再出现 `[LanguageConfig]` 日志;单测通过。

### 1.4 问题 1d:条件广播(影子副本比对)

**改动点**

`MarkdownReviewService`:
- `track()` 的 `documentChanged`(:394-428)在 1.2 改造中已按影响分级——`NONE` 不通知,天然消灭"改 B 文件、A/C 预览陪跑"的主来源(service 监听器只在批注状态真变化时被触发)。
- `reconcile`(:322-361)现有 `modified` 才 `changed()` 的逻辑保留,并比对 `start/end` 变化项(已有)。
- `changed()`(:570)本身保持;补充防御:通知循环里 listener 抛异常不得中断其余面板(单面板异常 try/catch + LOG.warn)。

**测试**:两个假 listener,触发 `changed()` 时其中一个抛异常,断言另一个仍被调用(现状会中断)。
**DoD**:改 B 文件时 A/C 预览无出站消息(冒烟计数)。

### 1.5 问题 12(部分):常量提取

**改动点**:新建 `com.github.claudecodegui.markdown.MarkdownReviewLimits`(final,私有构造):

```
MAX_PENDING_BATCHES=200, RESPONSE_CACHE=128, REQUEST_MAX_CHARS=256_000,
BATCH_MAX_CHARS=200_000, TEXT_MAX_CHARS=32_000, SPAN_MAX_CHARS=64_000,
DOC_MAX_CHARS=500_000, REQUEST_ID_MAX=128, TRANSFER_IDS_MAX=1000
```

替换 `MarkdownReviewService`(:500/:507 的 200、:474-511 各上限)与 `NativeMarkdownAnnotations`(:114/126/174/230/318/350)中的字面量。**不**暴露为用户配置(拍板)。

**测试**:现有行为不变(编译期替换);后续 PR-2 的兜底测试引用同一常量。

### 1.6 deprecated 顺手修(PR-1 范围)

- `NativeMarkdownAnnotations` 构造器 `addDocumentListener` → 带 Disposable 重载(见 1.1)。
- `ReadAction.run(ThrowableRunnable)` 两处(publish/handle)→ `ReadAction.compute(Computable<T>)` / `runReadAction(Computable)`,与文件内既有风格统一(:75、:184、:189 已是 Computable 风格)。
- `Messages.showChooseDialog` 留给 PR-4(对话框重构时一并换非废弃 API)。

**DoD(PR-1 汇总)**:verifier(262+233)deprecated 列表中 `NativeMarkdownAnnotations` 相关仅剩 PR-4 将处理的对话框项;`checkstyleMain`、全量 markdown 单测、前端 `npm test` 通过;冒烟三条新增断言(关闭零消息、NONE 零消息、跨文件零扇出)。

---

## 2. PR-2:草稿生命周期(问题 2 三层防御 + 问题 10)

### 2.1 Layer 1:丢弃方回执(含回执协议修正)

**关键发现(实施前提)**:现有回执按"**当前**窗口身份"匹配——`ClaudeChatWindow` 处理 `artifact_review_ack` 时(ClaudeChatWindow.java:2313-2328)用的是当前 `reviewTargetId` 和当前 `runtimeSessionEpoch`;前端回执 payload 也不携带 `targetId`(useArtifactReviews.ts:22-27)。换会话后,旧身份的回执**永远匹配不上**。必须先改协议:

1. `useArtifactReviews.ts` `acknowledge()`:payload 增加 `targetId: batch.targetId`。
2. `bindTarget`(:35-45):检测到 target 变化时,**先**对旧 `batchesRef.current` 逐批 `acknowledge(batch,'removed')`(回执自带旧 targetId/epoch),再清空并更新 `targetRef`。
3. `reset()`(:114-120):清空前同样逐批回执 `removed`。
4. `ClaudeChatWindow.java` ack 分支(:2313):改为**按 payload 的 targetId/targetEpoch 匹配**:`service.acknowledge(ack.targetId, ack.targetEpoch, ack.id, status, ids)`,不再与当前窗口身份绑定(仍校验 status 合法与 JSON 结构)。

### 2.2 Layer 2:收件人消亡即清理

1. `MarkdownReviewService` 新增 `public synchronized int dropPending(String targetId)`:移除该 targetId 全部 pending/received 批次(含 markers),返回条数。
2. 调用点一:**关闭聊天标签**。`ClaudeSDKToolWindow` 标签移除处(:364-379,调 `window.dispose()` 前)与 `disposeProjectChatWindows`(:203)调用 `dropPending(window.getReviewTargetId())`(判空、`project.isDisposed()` 防护)。
3. 调用点二:**换会话**。`ClaudeChatWindow.setSession` 回调(:358-369)在重新生成 `reviewTargetId`(:363)**之前**保存旧值,随后 `dropPending(旧值)`——旧身份的批次无人认领,保留即孤儿;persist 开着时批注本体仍在,可随时重新"加入 CC GUI"(拍板依据)。

### 2.3 Layer 3:配额兜底

`MarkdownReviewService.saveBatch`(:500-509)淘汰策略改为三级:

1. 优先淘汰最旧的非 pending 批次(现状);
2. 全 pending 时,优先淘汰 **targetId 不在存活窗口集合**的最旧 pending;
3. 仍无法判定(全 pending 且全存活)→ 淘汰插入序最旧的 pending。

存活集合的注入方式(避免存储层反向依赖 UI 层,遵循该类"core 不依赖 UI 类型"的架构注释):`MarkdownReviewService` 增加可注册字段 `private java.util.function.Supplier<Set<String>> liveTargetIds = () -> Set.of();`,`ClaudeSDKToolWindow` 初始化时注册 `() -> getChatWindows(project).stream().map(ClaudeChatWindow::getReviewTargetId).collect(toUnmodifiableSet())`。测试环境不注册 → 视为全灭 → 第 2 级直接生效。

超限异常改走问题 7 的错误编号(`LIMIT_REACHED`),不再上屏英文裸串。

### 2.4 persist 开关文案(拍板第 2 条)

`markdownReview.settings.persist.hint` 追加:"关闭后,项目关闭或 IDE 重启会丢失当前批注与未发送草稿"(en/zh 本 PR 落,其余 8 语言 PR-4 统一补)。

### 2.5 问题 10:先验证后清理

`useSubmitHandler.ts`(:136-148)调整顺序——清理动作全部后移到身份校验之后:

```
setTimeout(() => {
  if (reviews && reviewSubmission?.isCurrent && !reviewSubmission.isCurrent()) return; // 输入与附件原样保留
  onSubmit(content, attachmentsToSend);
  reviewSubmission?.onSubmitted();
  clearInput();
  if (externalAttachments === undefined) { setInternalAttachments([]); clearAttachmentsDraft?.(); }
}, 10);
```

- `close completions`、`recordInputHistory`、`invalidateImagePastes` 保持提交前执行(不依赖校验结果)。
- isCurrent 失败分支**静默保留**(内容还在输入框,用户重发即可),不弹 toast、不发 `onSubmitted`(批次保持 pending,等 Layer 1/2 收尾——协议已闭环)。
- 注意注释同步:`clearInput` 与"防抖回填"的现有注释(:133-135)随迁移更新。

### 2.6 测试与 DoD

**Java 单测**(MarkdownReviewServiceTest):
- 固化评审者的复现用例:写入 200 个不同 targetId 的 pending → `saveBatch` 不抛错、最旧一条被淘汰、计数仍为 200(三级淘汰第 2 级);
- 全存活 pending 满额 → 第 3 级淘汰;
- `dropPending` 计数与残留断言;terminal 批次不被 drop。
**前端单测**:
- `bindTarget` 变化 → 旧批次逐批 `artifact_review_ack`(status=removed,payload 含旧 targetId);
- `reset()` 同理;
- `useSubmitHandler`:mock `isCurrent=false` → `clearInput` 未被调用、`onSubmit` 未被调用、输入保留。
**冒烟**:`--review-policy` 场景扩展——切会话 ×N 后 `saveBatch` 正常;关标签后批次清零。
**DoD**:200 孤儿用例由红转绿;ack 按 payload 身份匹配后,换会话场景批次终态全部非 pending。

---

## 3. PR-3:行为语义(问题 3 选 A + 问题 9)

### 3.1 问题 3:去除 persist 翻转

1. `MarkdownReviewService.complete()`(:304-307):删除 `state.config.persist = true;` 及其注释;完成状态仅存内存,遵循用户 persist 选择。
2. `MarkdownCompletionTest` 重写断言:`complete` 后 `assertFalse(getConfig().persist)`;内存中 `completed=true` 可查询;`persist=false` 时 `getState()` 不含 payload、重启(`loadState`)后批注不恢复(既有行为,补显式断言)。
3. 设置页 persist hint 已在 PR-2.4 说明后果,无额外文案。

### 3.2 问题 9:删除/完成时撤回未发送的自动草稿

**范围(拍板)**:只撤 `automatic=true` 且 `kind=review` 的批次中的对应条目;引用(quote)与手动加入(transfer)的批次**不动**;已删除或已完成的条目都撤;发送侧再做防御性过滤。

**改动点**

1. `MarkdownReviewService` 新增:
```
/** 从该目标未决的自动批次中移除指定批注;批次空则转 removed。返回受影响批次。 */
public synchronized List<ReviewBatch> retractAuto(String targetId, String epoch, String annotationId)
```
   实现:遍历 `batches`,匹配 `targetId+epoch+isPending+automatic+kind=review`,从 `annotations` 移除该 id;空 → `state="removed"`;返回更新后的批次**拷贝**。不触发 `changed()`(草稿批次不属于预览状态)。
2. `NativeMarkdownAnnotations.handle()` 的 `"delete"` / `"complete"` 分支(:198-221):操作成功后,若 `service.getConfig().autoAddToChat`,对被操作批注调 `service.retractAuto(...)`——`targetId/epoch` 从批注最近一次交接记录取(需记录:交接成功时在 Annotation 上存 `lastTargetId/lastTargetEpoch`,或 service 侧按 annotationId 反查 pending 批次——**取反查方案**:新增 `List<ReviewBatch> pendingBatchesContaining(String annotationId)`,避免给 Annotation 加可变交接字段)。
3. 撤回的前端落地:`retractAuto` 返回的每个批次,经其 `targetId` 找到对应 `ClaudeChatWindow`(遍历 `ClaudeSDKToolWindow.getChatWindows(project)` 匹配 `getReviewTargetId()`),新增窗口方法 `removeArtifactReviewAnnotation(batchId, annotationId)` → `callJavaScript` 调前端 `window.removeArtifactReview(batchId, annotationId)`;前端 `useArtifactReviews` 已有 `remove()`(:85-96),自动回执 `removed/received`。
4. 防御性发送过滤(双保险的诚实边界):批次内 Annotation 冻结快照自带 `completed` 字段;`serializeArtifactReviews`(useArtifactReviews.ts:18)过滤 `annotation.completed===true` 的条目——只能拦住"冻结时刻已完成"的(交接时已过滤大部分),真正的保障是 3 的撤回链路。文档与代码注释如实说明。

### 3.3 测试与 DoD

- 单测:`retractAuto` 三态(移除后非空=received、空=removed、非 automatic/quote 批次不动);`pendingBatchesContaining` 查询。
- 冒烟:auto-add 场景——添加→删除,断言聊天草稿卡片消失;添加→完成,同断言。
- DoD:`MarkdownCompletionTest` 新断言通过;"已删除要求仍发给 AI"的 e2e 用例转绿(前端 `npm test`,现有 chat-input spec 扩展)。

---

## 4. PR-4:交互与文案(问题 5、6、7、11)

### 4.1 问题 5:interim 回执 + 未决幂等

**interim 协议**(消灭"选得慢 → 假超时 → 可重复点击"):

1. `NativeMarkdownAnnotations.transfer()`(:262-333):进入 `toolWindow.show(...)` 回调、检测到将弹选择框(`!automatic && windows.size() > 1`)时,先发中间回执:`{"requestId", "kind":"interim", "messageKey":"reviewTransferring"}`,再弹 `Messages.showChooseDialog`。
2. 前端 `index.ts` `send()`(:88-104)响应分支:收到 `kind==="interim"` → **清除现有 8s 计时器,重设为 5 分钟**,状态条显示 `labels.transferring`("正在等待你选择聊天…"),promise 保持挂起、按钮保持禁用(`pending` 不置 false);终态响应到达按现有逻辑 resolve。
3. 终态路径全部经 `remember()/failTransfer()` 已有 requestId 通道,无需改动。

**未决幂等**(兜底:interim 丢失/前端旧脚本时防重复):

4. `NativeMarkdownAnnotations` 新增 `Map<String,String> intentInFlight`:键 = 意图签名(panelId+action+start+end+selectedText+comment+documentVersion 的 SHA-256),值 = requestId。`handle()` 处理 create/quote 前先查:命中 → 直接复用 `results.get(在飞 requestId)`(若已有结果)或忽略本请求(在飞中,前端已有 interim 展示);transfer 完成/失败时清除签名。注:依赖 262 可用的摘要 API 不可假设——沿用现有 `hash()`(:360-366)的 `MessageDigest` 即可。
5. `Messages.showChooseDialog` deprecated 替换:实现时按 262 SDK 提示换非废弃选择对话框(候选:`ChooseFromListDialog` / `Messages.showEditableChooseDialog`,以 verifier 不再报该项为准)。

### 4.2 问题 6:畸形/超大请求立即回错

`NativeMarkdownAnnotations.receive()`(:113-131):

1. `raw.length() > 256_000` → 构造 `{"status":"error","errorKey":"BAD_REQUEST","detail":"too large"}` 立即 reply(requestId 不可知,字段缺省)+ `LOG.warn`(截断前 200 字符)。
2. JSON 解析失败 → 同上(能解析出 requestId 则带上)。
3. 字段校验失败(:123-127)但 requestId 可读 → 回 `errorKey=BAD_REQUEST`。
4. 前端 `labels.ts` 新增 `badRequest` 文案(10 语言,见 4.4)。

### 4.3 问题 7:错误编号化(用现有词典,拍板)

1. 新建包内枚举 `ReviewError`:`DISABLED, PREVIEW_CHANGED, DOC_CHANGED, SELECTION_MISMATCH, COMMENT_EMPTY, NOT_FOUND, REVISION_CONFLICT, LIMIT_REACHED, TOO_LARGE, TARGET_UNAVAILABLE, BAD_REQUEST, ID_LIST_INVALID`。每处 `throw new IllegalStateException/IllegalArgumentException(英文)`(NativeMarkdownAnnotations.java:159-245、MarkdownReviewService.java:216/277/288/301/319/508 等)改为携带 `ReviewError` 的异常(如 `ReviewOperationException(ReviewError.NOT_FOUND, args…)`)。
2. `handle()` 的 catch(:242-245):响应写 `errorKey`(枚举名)+ `args`(参数数组,供插值),**不再**写 `message` 英文原文。`failTransfer` 同步改造。
3. `orElseThrow` 裸错误(:202-203、:212-213,即计划 5c)随本项一并消灭:改 `findFirst().orElse(null)` + `throw ReviewOperationException(NOT_FOUND)`。
4. 前端 `index.ts` 响应分支:`errorKey` → `labels.errors[key]` 按 `args` 插值;`labels.ts` 为 10 语言补 `errors` 映射表(以现有 `previewLabels` 的翻译风格人工校对)。

### 4.4 问题 11:8 种语言全量补齐

1. 为 `es/fr/hi/ja/ko/pt-BR/ru/zh-TW` 八个 locale 文件补 `markdownReview.*` 全部键(以 en 为源;术语对照 `labels.ts` 既有译文风格;**人工校对,禁机翻直接入库**)。
2. 新增键统一收录:enabled.hint 备注(1a)、persist.hint(2.4)、`transferring`、`badRequest`、`errors.*`(4.1–4.3)。
3. 键完整性校验:新增 `webview/src/i18n/markdownReview.test.ts`——遍历 10 个 locale,断言 `markdownReview` 子树键集合与 en 完全一致(模式参考现有 `workspaceDiff.test.ts`)。

### 4.5 测试与 DoD

- 单测:`ReviewError` 全枚举 → 前端 `labels.errors` 10 语言全覆盖断言(键完整性测试同时覆盖);`receive` 三类畸形输入即时回错(pipe mock);interim 重置计时器(前端单测:fake timers)。
- 冒烟:多标签 + 延时 >8s 选聊天 → 不出现超时误报、无重复草稿;quote 连点无重复。
- DoD:verifier(262+233)deprecated 列表清零 `Messages.showChooseDialog` 项;预览浮层任何操作失败不再出现英文裸串(遍历冒烟场景);i18n 完整性测试进 `npm test`。

---

## 5. PR-5:流程收尾

1. **233 校验目标提交**:build.gradle `ides` 块的 `ide('IC','2023.3.7')`(+3 行,工作区已有)随本 PR 提交;`docs` 或 README 补充 verifyPlugin 重跑规程:**重跑前删除 `build/reports/pluginVerifier/*/report.html`**(校验器 HTML 报告器对旧文件抛 FileAlreadyExistsException,退出码 1 与结论无关;结论以各 IDE 目录 `verification-verdict.txt` 为准)。
2. **技术债登记**:开发文档记录——`MarkdownBrowserPreviewExtension` 标 `@ApiStatus.Obsolete`(243/262 均是)、`MarkdownHtmlPanel` 访问器 `@ApiStatus.Experimental`;升级复核清单:每次 IDE 大版本升级跑 verifyPlugin 双目标 + 打开预览冒烟。
3. **PR 瘦身**(计划 5f):`docs/plans/2026-10-09-native-markdown-annotations-development-plan.md`(410 行)与 `scripts/native-markdown-runtime-smoke.mjs`(607 行)、`scripts/native-markdown-smoke.init.gradle` 移出主 PR——迁至本 docs 分支(与本文档同模式),`docs/markdown-review-pr.md` 已被忽略规则覆盖,保持。
4. **问题 12 全库复查**(拍板):批注功能全部改动文件 walkthrough,六类清单逐项过:写死上限、无界增长、满额无兜底、静默回退、硬编码文案、设置被隐式改写;发现即追加到计划文档并按需开小 PR。
5. **收尾构建**:`gradlew test checkstyleMain buildPlugin verifyPlugin -PskipWebview=true` 双目标绿;`webview npm test` + `npm run build` 绿。

---

## 6. 全局验收清单

- [ ] enabled=false:预览打开 + 按键,出站消息 0(首帧精简状态除外)
- [ ] 有批注文件打字:NONE 编辑零消息;SHIFT/HIT 每编辑窗 ≤ 个位数轮
- [ ] 改 A 文件,B/C 预览零出站消息
- [ ] 200 孤儿:saveBatch 不再抛错,兜底淘汰生效
- [ ] 换会话/关标签:批次全部转 removed/dropPending 清零
- [ ] complete 后 persist 保持 false
- [ ] auto-add 下删除/完成:聊天草稿同步消失
- [ ] 提交瞬间换会话:输入保留、内容未发、批次不悬空
- [ ] 多标签慢选聊天:无超时误报、无重复草稿/批注
- [ ] 预览报错:10 语言,无英文裸串,无 "No value present"
- [ ] i18n:10 个 locale `markdownReview` 键集合一致
- [ ] verifier 262+233:双目标 Compatible,deprecated 清单无本功能残留项
- [ ] 全库六类复查完成,结论记录

## 7. 风险与回滚

- 每个 PR 独立可回滚;PR-1 触碰推送协议(`publish` 频率/精简状态)与前端 dirty 展示节奏——前后端同包发布,无兼容窗口;若冒烟发现 dirty 期间交互退化,回滚点集中在 1.2 的 `documentChanged` 改动。
- PR-2 的 ack 协议改动(payload 加 targetId、按 payload 匹配)与 PR-3 撤回链路有耦合,故 PR-3 依赖 PR-2 先合。
- PR-4 的 interim 协议为新增响应类型,旧前端不识别 interim 会忽略并维持 8s 超时——同包发布后不存在该窗口。

---

## 附录 A:六项关键设计的代码级详稿

以下为评审讨论中标记的 6 项设计决策的展开稿。代码块为**设计形态**(表达结构与意图),非最终实现;`//` 注释说明与现状的差异。

### A.1 1b:影响分级 + 事件驱动推送

**现状回顾**:面板自身的 DocumentListener 每键全量 `publish()`;服务 `track()` 注册的 DocumentListener 每键做受影响清理 + marker 位置同步 + **无条件** `changed()` → 所有面板再各 publish 一次。前端 afterPatch 还会回发 `sync` 触发 reconcile+publish——一次按键 ≈ 2 轮全量推送 + 1 次全文核对。

**新结构**:

```java
// MarkdownReviewService 新增
public enum EditImpact { NONE, SHIFT, HIT }

// 订阅 API:Runnable → Consumer<EditImpact>(调用方仅 NativeMarkdownAnnotations 与单测,迁移可控)
private final List<java.util.function.Consumer<EditImpact>> listeners = new CopyOnWriteArrayList<>();
```

```java
// track() 的 documentChanged(替换现有 :394-428 尾部)
@Override
public void documentChanged(@NotNull DocumentEvent event) {
    boolean hit = !affected.isEmpty();              // beforeDocumentChange 已算好(含 preservesSource 例外)
    synchronized (this) {
        if (state.config.enabled && state.config.autoDeleteAffected && hit) {
            /* 现有受影响清理,不变 */
        }
        affected.clear();
        EditImpact impact = classify(event, hit);   // 见下
        if (impact == NONE) return;                 // ← 1d:无影响零通知,跨文件扇出自然消失
        /* SHIFT/HIT:marker 位置同步循环保持现状(供随后的轻量/全量发布使用) */
    }
    changed(impact);
}

static EditImpact classify(DocumentEvent event, boolean hit) {
    if (hit) return HIT;                            // 命中批注区间
    if (annotationsOfThisDocument.isEmpty()) return NONE;
    int editStart = event.getOffset();
    boolean anyAfter = 存在批注 annotation.start >= editStart;
    return anyAfter ? SHIFT : NONE;                 // 编辑点在全部批注之后 → 谁都不动
}
```

```java
// NativeMarkdownAnnotations
private final MarkdownReviewService.Consumer listener = impact -> publish(impact != HIT);
//   ↑ SHIFT → publish(light);HIT → publish(full)
private final DocumentListener documentListener = new DocumentListener() {
    @Override public void documentChanged(@NotNull DocumentEvent event) { dirty = true; }  // ← 不再 publish
};
//   注册改带生命周期归属:document.addDocumentListener(documentListener, service)  ← deprecated 顺手修
```

```java
// publish(boolean light)
private void publish(boolean light) {
    /* 门禁(1a)与 enabled 精简短路在前 */
    ReadAction.run(() -> {
        state.addProperty("documentVersion", document.getModificationStamp());
        state.addProperty("dirty", dirty);
        /* annotations 照发(位置已由 marker 同步,轻量路径也发——高亮要跟着动) */
        state.add("annotations", GSON.toJsonTree(service.list(file)));
        if (!light) {                               // light 跳过全文 Mermaid 扫描
            state.add("diagrams", GSON.toJsonTree(RenderedSelectionMatcher.mermaidRanges(document.getText())));
        }
        reply(state);
    });
}
```

**关键决定**:

1. **时间窗不自建**——前端 afterPatch 的 `sync` 就是合并节拍:后端不再每键主动推,一次编辑 = 一个渲染补丁 = 一次 sync = 一次 reconcile+publish。备选(实测 sync 仍高频时):`handle("sync")` 入口加 `Alarm` 100ms 合并(`cancelAllRequests + addRequest(doSync, 100)`,`ready` 不合并)。
2. **面板自身 listener 保留但只置 dirty**:无批注文档没有服务级监听,面板 listener 是其唯一的 dirty 信号(创建门禁依赖)。
3. dirty 传到前端的时机从"按键瞬间"变为"下一次 sync/操作"——间隔为 IDE 渲染节拍;`handle("create")` 的 documentVersion 校验仍兜底。冒烟列为观察项。

**测试**:`classify` 表驱动(命中/批注前/批注后/空集/删除跨越);service 级 NONE 零通知、SHIFT→light、HIT→full(计数 listener);listener 异常不中断广播(try/catch + warn)。

### A.2 问题 2 Layer 1:回执按"批次自身身份"匹配

**现状缺陷**:`acknowledge` payload 不带 `targetId`;Java 端用**当前窗口**的 reviewTargetId/epoch 匹配,还要求 payload epoch == 当前 epoch(ClaudeChatWindow.java:2320-2323)。→ 换会话后,对旧批次的一切回执**必然失配**,这是孤儿链的协议级根因。

**协议改动(3 处)**:

```ts
// useArtifactReviews.ts — payload 增加 targetId
function acknowledge(batch: ArtifactReviewBatch, status: 'received'|'queued'|'removed') {
  return sendBridgeEvent('artifact_review_ack', JSON.stringify({
    id: batch.id,
    targetId: batch.targetId,        // ← 新增:批次冻结的身份
    targetEpoch: batch.targetEpoch,
    status,
    annotationIds: batch.annotations.map(a => a.id),
  }));
}

// bindTarget:换目标时先逐批回执 removed,再清空
const old = targetRef.current;
if (old && (old.targetId !== target.targetId || old.targetEpoch !== target.targetEpoch)) {
  for (const batch of batchesRef.current) acknowledge(batch, 'removed');   // ← 旧身份回执
  batchesRef.current = []; setBatches([]);
}
targetRef.current = target;

// reset():清空前同样逐批回执 removed
```

```java
// ClaudeChatWindow — artifact_review_ack 分支(:2313-2328)
String ackTargetId = ack.get("targetId").getAsString();   // ← 用回执自带的身份
String ackEpoch = ack.get("targetEpoch").getAsString();
MarkdownReviewService.getInstance(project)
    .acknowledge(ackTargetId, ackEpoch, ack.get("id").getAsString(), status, ids);
// ← 删除"payload epoch 必须等于当前会话 epoch"的校验:回执是历史事实,不属于当前状态
```

- 服务端 `acknowledge()`(MarkdownReviewService.java:535-549)**零改动**——它本来就按参数三元组精确匹配,一直只是调用方传错身份。
- 顺序安全:回执读取的是 batch 自带的 targetId/epoch,与 `targetRef` 无关。
- 安全边界:伪造回执需要知道目标批次 id(与现状同量级威胁),不新增校验。

### A.3 问题 2 Layer 3:配额兜底三级淘汰 + 存活集合注入

```java
// MarkdownReviewService
private java.util.function.Supplier<Set<String>> liveTargets = Set::of;  // 默认"全灭"→ 测试/无 UI 环境直接走第②级
public void setLiveTargetsSupplier(java.util.function.Supplier<Set<String>> supplier) { liveTargets = supplier; }

private ReviewBatch pickVictim() {                 // 替换 :500-506 的单级淘汰
    Set<String> alive = liveTargets.get();        // 调用线程:EDT(saveBatch 链路)
    ReviewBatch victim = state.batches.stream().filter(b -> !isPending(b)).findFirst().orElse(null); // ① terminal,现状
    if (victim == null) victim = state.batches.stream()
            .filter(b -> isPending(b) && !alive.contains(b.targetId)).findFirst().orElse(null);      // ② 已消亡目标
    if (victim == null) victim = state.batches.stream().filter(this::isPending).findFirst().orElse(null); // ③ 最旧
    return victim;                                 // "最旧"=列表序(batches 为 append-only,序即时间序)
}

// saveBatch 淘汰段
while (state.batches.size() >= MarkdownReviewLimits.MAX_PENDING_BATCHES) {
    ReviewBatch victim = pickVictim();
    if (victim == null) break;
    state.batches.remove(victim);
}
if (state.batches.size() >= MarkdownReviewLimits.MAX_PENDING_BATCHES) {
    throw new ReviewOperationException(ReviewError.LIMIT_REACHED);
}
```

```java
// ClaudeSDKToolToolWindow 初始化时注册(避免存储层反向 import UI 类)
MarkdownReviewService.getInstance(project).setLiveTargetsSupplier(() ->
    getChatWindows(project).stream().map(ClaudeChatWindow::getReviewTargetId)
        .collect(java.util.stream.Collectors.toUnmodifiableSet()));
```

- **语义**:淘汰只删"交接记录",批注本体仍在(annotations 独立持久化),用户可重新"加入 CC GUI"——符合拍板依据。
- 兜底是补漏设计:Layer 1(A.2)+ Layer 2(dropPending)正常工作时永远走不到 ②/③。

### A.4 问题 9:删除/完成的草稿撤回链路(复用前端 remove 通道)

**链路**:

```
预览点"删除/标记完成"
 → handle("delete"/"complete") → service.update/complete(批注本体,现有)
 → [新] service.retractAutoByAnnotation(annotationId)          // 反查未决 automatic 批次
 → 对每个受影响批次:按 batch.targetId 找窗口
    → window.removeArtifactReviewAnnotation(batchId, annotationId)   // 新窗口方法
       → callJavaScript("removeArtifactReviewAnnotation", …)
          → 前端 window.removeArtifactReviewAnnotation = (b,a) => remove(b,a)
             → 现有 remove():过滤条目 + acknowledge('received'/'removed')  ← 回执自动闭环
```

```java
// MarkdownReviewService 新增
/** 从含该批注的未决 automatic review 批次中移除它;批次空则转 removed。返回受影响批次拷贝。 */
public synchronized List<ReviewBatch> retractAutoByAnnotation(String annotationId) {
    List<ReviewBatch> touched = new ArrayList<>();
    for (ReviewBatch batch : state.batches) {
        if (!isPending(batch) || !batch.automatic || !"review".equals(batch.kind)) continue;  // 引用/手动不动(拍板)
        boolean removed = batch.annotations.removeIf(a -> annotationId.equals(a.id));
        if (!removed) continue;
        if (batch.annotations.isEmpty()) batch.state = "removed";
        touched.add(copy(batch, ReviewBatch.class));
    }
    return touched;
}
```

```java
// NativeMarkdownAnnotations.handle() 的 delete/complete 分支尾部
if (service.getConfig().autoAddToChat) {
    for (ReviewBatch batch : service.retractAutoByAnnotation(annotationId)  /* id 取自请求 */) {
        ClaudeChatWindow window = ClaudeSDKToolWindow.getChatWindows(project).stream()
                .filter(w -> batch.targetId.equals(w.getReviewTargetId())).findFirst().orElse(null);
        if (window != null) window.removeArtifactReviewAnnotation(batch.id, annotationId);
        // 窗口不在(目标已消亡)→ 无需撤:前端看不到它,Layer 2 会 dropPending
    }
}
```

- 前端接线一行:`window.removeArtifactReviewAnnotation = (batchId, annId) => remove(batchId, annId);`(useArtifactReviews.ts 的 useEffect 内,与 addArtifactReview 并列注册/注销)。
- **时序安全**:前端 `remove()` 幂等(条目不存在 = no-op + 'received' 回执);用户同时在聊天区手动删 = 两次过滤,结果一致。
- reopen(取消完成)不恢复草稿——草稿是"未发送快照",reopen 后可重新 transfer,文案注明。

### A.5 问题 5:interim 协议 + 意图签名幂等

**interim(主修复——顺带掐断"超时后可再点")**:

```java
// NativeMarkdownAnnotations.transfer(),弹框前
if (!automatic && windows.size() > 1) {
    JsonObject interim = new JsonObject();
    interim.addProperty("requestId", requestId);
    interim.addProperty("kind", "interim");
    interim.addProperty("messageKey", "reviewTransferring");
    reply(interim);                                  // ← 模态框弹出之前
    /* 现有 Messages.showChooseDialog(该 API 的非废弃替换在 4.1 一并做) */
}
// interim 不写入 results 缓存(不应被重放);终态照常 remember()+reply()
```

```ts
// index.ts send() 的响应分支
if (parsed.kind === 'interim') {
    clearTimeout(entry.timer);
    entry.timer = setTimeout(() => {                  // 兜底阈值 5 分钟
        callbacks.delete(requestId); message = labels.timeout; pending = false; render(); resolve(false);
    }, 300_000);
    message = labels.transferring;                    // 状态条:"正在等待选择聊天…"
    render();
    return;                                           // 不 resolve → pending 保持 true → 按钮仍禁用
}
```

**意图签名幂等(兜底,防 interim 丢失/旧脚本)**:

```java
// NativeMarkdownAnnotations 字段
private final Map<String, String> inFlightIntents = new LinkedHashMap<>() { /* 超过 64 清最旧 */ };

// handle() 的 create/quote 入口(校验之后、执行之前)
String signature = hash(panelId + "|" + action + "|" + start + "|" + end + "|"
        + selected + "|" + comment + "|" + document.getModificationStamp());
String owner = inFlightIntents.get(signature);
if (owner != null) {
    JsonObject done = results.get(owner);
    if (done != null) { reply(done); return; }        // 前一请求已有终态 → 重放
    reply(interimOf(requestId)); return;              // 仍在飞 → 回 interim,不重复创建
}
inFlightIntents.put(signature, requestId);
/* 执行;remember()/failTransfer() 时 inFlightIntents.values().remove(requestId) */
```

- 签名含 `documentVersion` → 文档变了就是新意图,不会错误去重;重放依赖现有 `results` LRU(128)。
- **防重复的两道闸**:interim 让按钮在等待期间不可点(主);签名幂等拦住同意图的并发/重试请求(兜)。误报场景由 interim 消灭,正常并发由签名拦截。

### A.6 问题 7:错误编号化(枚举 → 词典)

**枚举 ↔ 现有抛出点对照(全部 throw 一次迁移)**:

| ReviewError | 现有英文串(出处) | args |
| --- | --- | --- |
| DISABLED | "…are disabled in CC GUI settings."(NativeMarkdownAnnotations:159) | — |
| PREVIEW_CHANGED | "Preview changed. Select the text again."(:162) | — |
| DOC_CHANGED | "Document changed. Wait for the preview…"(NativeMarkdownAnnotations:168) | — |
| SELECTION_MISMATCH | :181(现为 messageKey,统一并入 errorKey) | — |
| COMMENT_EMPTY | service:216、:288 | — |
| NOT_FOUND | service:319 + 两处 orElseThrow(:203、:213,即计划 5c) | id |
| REVISION_CONFLICT | service:278、:301 | — |
| LIMIT_REACHED | service:508(A.3 起改抛枚举) | — |
| TOO_LARGE | NativeMarkdownAnnotations:174、:319 | 限值 |
| TARGET_UNAVAILABLE | failTransfer 四条(:279-307) | detail(仅入日志) |
| ID_LIST_INVALID | :231 | — |
| BAD_REQUEST | 问题 6 新增三类畸形请求 | — |

```java
// 公共异常:service 与 annotations 共用;仍是 RuntimeException,调用方/单测签名不变
public class ReviewOperationException extends RuntimeException {
    public final ReviewError error; public final Object[] args;
}

// handle() catch(NativeMarkdownAnnotations:242-245)
} catch (ReviewOperationException error) {
    response.addProperty("status", "error");
    response.addProperty("errorKey", error.error.name());
    response.add("args", GSON.toJsonTree(error.args));   // 不再写 message 英文原文
} catch (RuntimeException error) {                          // 兜底:未归类
    response.addProperty("status", "error");
    response.addProperty("errorKey", "INTERNAL");
    LOG.warn("Unmapped review operation failure", error);
}
```

```ts
// labels.ts:浮层自带的 10 语言词典(不进 i18next——浮层是独立 bundle,labels.ts 本就是它的翻译层)
errorKeys: { DISABLED: '设置中已关闭 Markdown 批注。', DOC_CHANGED: '文档已变化,请等预览更新后重新选择。',
             NOT_FOUND: '批注 #{args} 已不存在。', /* …12 键 ×10 语言 */ }
// index.ts 响应分支
message = parsed.errorKey && labels.errorKeys[parsed.errorKey]
    ? labels.errorKeys[parsed.errorKey].replace('{args}', String(parsed.args?.[0] ?? ''))
    : labels.failed;
```

**完整性测试**:枚举名集合 ↔ 10 份 labels.errorKeys 键集合双向比对(前端单测,与问题 11 的 locale 完整性测试互为补充);冒烟遍历失败场景断言无英文裸串、无 "No value present"。
