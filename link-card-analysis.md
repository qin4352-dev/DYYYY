# DouyinHelper.dylib「链接卡片」功能反汇编分析报告

- 目标：`DouyinHelper.dylib`（Mach-O arm64，18,192,944 字节，sha256 79d6cf78…），作者 Xuu 的抖音注入 tweak
- 分析方式：手工解析 ObjC 元数据 + capstone 反汇编 + 字符串/选择子交叉引用（工具链同评论颜色分析）
- 原始 dump 同目录：`disasm_linkcard_*.txt`（各方法反汇编）、`xrefs_linkcard.txt`（交叉引用点）、`xref_linkcard.py`

## 结论先行

**链接卡片走私信（IM）通道，与普通文本私信完全同一条发送漏斗；卡片本质是伪造抖音原生的「H5 分享卡片」消息：消息类型常量 26，内容是一个固定键结构的字典。** 构造方式：用抖音 IM SDK 的 `IESIMSendMessageModel`（`initWithConversationId:` 建模、`setMessageType: 26`、`setMessageContentDict:` 装卡片字典），再交给 `IESIMMessageSender` 的 `asyncSendMessage:completion:`（首选）或 `sendMessage:`（兜底）发出。不依赖任何抖音头文件——类全靠 `NSClassFromString` 按名取，方法靠动态选择子调用。

触发方式共三条（全部实证，见下文）：
1. **聊天输入口令**：在任意私信会话里发送文本 `#卡片`（发当前选中的卡）或 `#卡片 <关键词>`（发关键词匹配到的卡），tweak hook 了抖音的发送漏斗，在消息出门前把这条文本消息**原地改造成**卡片消息；
2. **关键词自动回复**：对方发来的消息命中预设关键词、且该条目绑定了卡片关键词时，自动把对应卡片回复到该会话；
3. **手动发送**：菜单动作把「当前选中卡片」发到「当前打开的会话」。

卡片数据存 NSUserDefaults：列表 key `WX_Card_List`（字典数组），单条卡片键为 `title / desc / url / thumb / keywords`；另有一套单卡槽位 `xuuz_title / xuuz_desc / xuuz_link_url / xuuz_cover_url / xuuz_keywords`。**`keywords` 字段不是搜索用，而是匹配键**：自动回复条目的 `cardKeyword` 和口令 `#卡片 <关键词>` 都靠它在卡片列表里找卡。

## 两个同名类是怎么回事（实证）

二进制里有两套几乎同名的发送实现，是**同一套代码复制了两份、挂在两个类上**，且配置来源不同：

| | `VoiceEomjiSend`（@0x10302b0） | `AutoMessageManager`（@0x102fbd0） |
|---|---|---|
| 定位 | 通用消息发送工具类（文本/音乐/语音表情/@全体/卡片都发），`+sharedInstance` @0xe19e0 | 「自动消息」批量引擎（每日任务、已发名单、禁发时段），`+sharedInstance` @0xa20ec |
| `sendLinkCardToConversation:` | IMP **0xe3f24** | IMP **0xa73b0** |
| `_internalSendLinkCard…` | IMP **0xe44f0** | IMP **0xa787c** |
| 卡片数据来源 | `WX_Card_List` + `WX_Selected_Card_Index`（卡片列表里选中的那张） | NSUserDefaults 单卡槽位 `xuuz_cover_url / xuuz_link_url / xuuz_title / xuuz_desc` |
| 内容字典 | **5 键**（多一个驼峰 `coverURL`） | **4 键**（只有 `cover_url`） |
| 实际调用方 | 自动回复引擎、菜单手动发送（见触发一节） | **未找到任何调用点**（见下） |

两份 `_internalSend…` 的构造逻辑逐指令同构（同样的类名、同样的 type=26、同样的错误域/文案），仅字典差一个键。`AutoMessageManager` 自己的定时批量任务（`_performDailyTaskAsync` @0xa2a3c → `sendSingleMessageAsync:toConversation:` @0xa2ecc）实际只调用 `_internalSendMessage:toConversation:` 发**文本**，不发卡片；全二进制对 `sendLinkCardToConversation:` 选择子的直接引用点只有一个（0x8f7c，接收者是 `VoiceEomjiSend`）。**推断：`AutoMessageManager` 那份卡片代码是复制后未接线的冗余副本**；但它的配置槽位 `xuuz_*` 并非死数据——卡片列表页选中卡片时、以及 `#卡片` 口令路径都读写这套槽位（见存储一节）。

两个类还各有 `fetchSDKRoot`（VoiceEomjiSend @0xe1b54 / AutoMessageManager @0xa26f4）和 `_sdkRoot`、`_sdkSemaphore` 字段：发送前先用信号量串行化，并要求 `sdkRoot != nil`，否则直接返回 NSError（域 `AutoMessage`、码 -1、描述「SDK 不可用」）。`sdkRoot` 是抖音 IM SDK 实例对象，来源有二（实证）：`fetchSDKRoot` 主动调 `[TIMXSDKInstancesManager iesim_TIMXSDKInstance]`（兜底 `iesim_TIMXSDKInstance_Nullable`）存入；构造函数 @0xa050 还 hook 了 `+[TIMXSDKInstancesManager iesim_TIMXSDKInstance]`（替换 IMP @0xa124）顺手把实例与**本机用户 ID** 捕获进全局（自身 ID 全局 @0x1508030，写入点 0xa1e0）。

## 通道与参数（问题 1、3）

- **通道：私信（IM 会话），没有评论路径。** 三条触发路径最终都落到同一个会话 ID 上发一条 IM 消息。
- **`conversation` 参数实为会话 ID 字符串**（实证，三处互证）：`_internalSend…` 把它原样传给 `-[IESIMSendMessageModel initWithConversationId:]`；日志格式串为 `[AutoMsg] 开始发送卡片 cid=%@`；自动回复路径传的是消息的 `belongingConversationIdentifier`；手动路径传的是全局捕获的当前会话 ID（全局 @0x1508010）。
- **发送漏斗与文本私信是同一个**（实证）：`VoiceEomjiSend` 的文本发送 `_internalSendTextMessage:text:` @0xe60a8 用同样的 `IESIMMessageSender` + `IESIMSendMessageModel`，仅 `setMessageType:` 传 **7**、内容字典换成文本结构（`text / scene / ai_ext="{}" / mention_users / related_share_video / richTextInfos`）。卡片与文本只差 type 与 content dict。
- 发送调用顺序（`_internalSend…` 实证）：`[[IESIMMessageSender alloc] init]` → 若响应 `asyncSendMessage:completion:`（`NSSelectorFromString` 动态查）就走异步发送，完成回调在主队列弹 toast（成功：`CustomToastView` 标题「消息」、副标题「分享卡片已发送 ✅」；失败：NSLog `[AutoMsg] 分享卡片发送失败 cid=%@`）；不响应则兜底 `sendMessage:`，返回 nil 时造 NSError（域 `AutoMessage`、码 -2、描述「发送失败」）。全程 `@try` 兜底：异常转 NSError 码 -3（描述取异常 `reason`）。

## 卡片构造过程（问题 2，`_internalSendLinkCardToConversation:coverURL:title:desc:linkURL:`）

参数序：x2=会话 ID、x3=coverURL、x4=title、x5=desc、x6=linkURL。步骤（VoiceEomjiSend 版 @0xe44f0 实证）：

1. `dispatch_semaphore_wait(_sdkSemaphore, -1)` 串行化；`sdkRoot == nil` → 报错返回（见上）。
2. `NSClassFromString(@"IESIMMessageSender")`、`NSClassFromString(@"IESIMSendMessageModel")`，任一为 nil → 信号量释放后静默返回 nil。
3. `model = [[IESIMSendMessageModel alloc] initWithConversationId:会话ID]`。
4. `[model setMessageType: 0x1a]` —— **消息类型常量 = 26**（两份实现一致，立即数实证）。
5. 组内容字典 `[NSDictionary dictionaryWithObjects:forKeys:count:]` 后 `[model setMessageContentDict:dict]`。字段映射：

   VoiceEomjiSend 版（5 键，count=5）：
   | 字典键 | 值 |
   |---|---|
   | `coverURL` | coverURL 参数 |
   | `cover_url` | coverURL 参数（同值双写，驼峰+下划线各一份） |
   | `title` | title 参数 |
   | `desc` | desc 参数 |
   | `link_url` | linkURL 参数 |

   AutoMessageManager 版（4 键，count=4）：只有 `cover_url / title / desc / link_url`，无驼峰 `coverURL`。

   任一参数为 nil 时以常量空串兜底（地址 0xff00a8 处的 __cfstring 结构，位于 __cfstring 数组内、步距 0x20，相邻成员为 'userID'/'nickname'/', '/'无'；本轮解析器漏录此条，**推断为 `@""`**）。
6. 按上节漏斗发送；`dispatch_semaphore_signal` 收尾；返回 NSError 或 nil。

注意：字典是**纯字典直塞** `setMessageContentDict:`，没有逐字段 setter、没有 JSON 序列化（JSON 只出现在备份功能里）。这说明抖音端对 type=26 的内容字典就是这个松散结构，键名对齐即可渲染成 H5 卡片。

## 三条触发路径（问题 4、6）

### 路径 A：聊天口令 `#卡片`（hook 抖音发送漏斗，原地改造）

构造函数 @0x10f94 实证注册：`-[IESIMMessageSender sendMessage:]` → 替换 IMP **0x14b60**（同 ctor 还 hook 了 `-[AWEIMNewMessageDataController sendMessage:mentionUsers:customToast:enterFrom:]` → 0x14a28，为另一入口的同类改造，本报告未逐行展开）。替换 IMP 的行为（实证）：

- 取待发模型的 `originSendMessage`，若是 `AWEIMTextMessage`（objc_getClass 按名校验）才处理；
- 开关：NSUserDefaults `boolForKey: @"xuuz_hookim_enable"` 为真才启用本路径；
- 文本恰好等于 `#卡片`：读单卡槽位 `xuuz_cover_url / xuuz_link_url / xuuz_title / xuuz_desc`（为空时的默认值为 cover `https://example.com/default_cover.jpg`、title「默认卡片」、desc「这是默认描述」、link `https://example.com/`），组 5 键字典（`cover_url / coverURL / title / desc / link_url`）；
- 文本以 `#卡片 `（带空格）开头：余下部分去空白后作为关键词，在 `WX_Card_List` 里找 `card[@"keywords"]` 满足 `containsString:` 的第一张卡（日志 `[HookIM] #卡片 %@ 匹配成功 %@`），用该卡的 `title / desc / url→link_url / thumb→cover_url+coverURL` 组 5 键字典；
- 改造手法：`h5 = [[AWEIMShareH5Message alloc] initWithContentDict:dict]`；`[model setOriginSendMessage: h5]`；`t = [h5 messageType] ? : 7; [model setMessageType: t]`；`[model setMessageContentDict: [h5 messageContentDict] ?: dict]`；打日志 `[HookIM] 替换 #卡片 文本 -> H5 卡片 title=%@ link=%@ type=%lld` 后放行原 `sendMessage:`。
  - **推断**：`AWEIMShareH5Message` 以内容字典初始化后其 `messageType` 即 26（与直发路径写死的 0x1a 互证），兜底值 7 只是取不到时的防御分支（7 是文本类型，见文本路径实证）。
- 同一个 hook 里还并列处理了 `#zyy `（TTS 语音）、`#日本骂人`、`#dj` 等其他口令（与卡片无关，说明这是作者的「口令总开关」hook）。

### 路径 B：关键词自动回复（收到消息时触发）

构造函数 @0xa050 实证注册一组 TIMXO 模型 hook，其中 `-[TIMXOMessage initWithRootObject:model:]` → 替换 IMP **0xa398** 是自动回复引擎（同组还有 `+[TIMXSDKInstancesManager iesim_TIMXSDKInstance]`、`-[TIMXOSendMessage initWithRootObject:]` @0xa244、`setContent:` @0xa2dc、`setMessageType:` @0xa358）。引擎逻辑（0xa398 起实证）：

- 调原实现拿到消息对象；NSUserDefaults `boolForKey: @"AutoReplyEnabled"` 关则透传；
- 经 `internalModelStore.sender` 取发送者，与全局本机 ID（@0x1508030）相等 → 跳过（不回复自己）；`localCreatedAt` 距今 > 1 秒 → 跳过（只理刚到的新消息，防历史消息重放触发）；
- 取 `belongingConversationIdentifier`（会话 ID）与 `content[@"text"]`（文本），日志 `[新消息]\n会话ID:%@\n内容:%@`；
- 三个平行数组 `AutoReplyKeywords / AutoReplyReplies / AutoReplyCardKeywords`（AutoReplyEditor 保存，见存储节）逐条比对：`[text containsString: keyword]` 命中后——
  - 该条 `cardKeyword` 非空：在 `WX_Card_List` 里找第一张 `card[@"keywords"] containsString: cardKeyword` 的卡 → `[[VoiceEomjiSend sharedInstance] sendCardToConversation:会话ID title:card.title desc:card.desc linkurl:card.url coverurl:card.thumb]`（调用点 @0xaacc，日志 `[AutoReply] 匹配卡片关键词:%@ 发送卡片:%@`；找不到卡日志 `[AutoReply] 未找到卡片:%@`）；
  - 否则走文本回复：`[[VoiceEomjiSend sharedInstance] sendTextMessageToConversation:会话ID text:reply]`（日志 `[AutoReply] 匹配关键字:%@ 回复:%@`）。
- `sendCardToConversation:title:desc:linkurl:coverurl:`（VoiceEomjiSend @0xe3c44）先校验 conversation/title/linkurl 非 nil（日志 `[AutoMsg] 准备发送卡片 cid=%@`），切主队列后调 `_internalSend…`（coverurl→coverURL、linkurl→linkURL），失败弹 toast「分享卡片发送失败 ❌ (%@)」。

即：**「关键词回复」与卡片是同一引擎的两个出口**——条目要么回文本、要么回卡片，由条目的 `cardKeyword` 决定；卡片编辑页的 `keywords` 字段就是被 `cardKeyword` 命中的那串匹配键。入口 UI 上也相连：`AutoReplyEditor`（关键词回复编辑器）里有 `- openCardConfig` @0xacd00，全文一条调用——`[WXCardListViewController show]`，即从回复编辑器直接跳卡片列表配置。

### 路径 C：手动发送选中卡（菜单动作）

C 函数 @0x8f40（与 `CopyConversationIDAndNicknameToPasteboard` 等菜单动作同簇）：`[[VoiceEomjiSend sharedInstance] sendLinkCardToConversation: *(id *)0x1508010]`，参数是全局「当前会话 ID」。该全局的写入点在 0xc094 与 0x13798（分别落在长按会话 cell 的 hook 函数区与聊天页 `viewDidAppear:` hook 函数区内；**写入点实证，归属函数的宿主类为推断**）。`VoiceEomjiSend sendLinkCardToConversation:` @0xe3f24 的取卡逻辑（实证）：

- 读 `WX_Card_List` 与 `WX_Selected_Card_Index`，越界/类型不对则卡片为 nil；
- 取卡的 `title / desc / url / thumb`（nil 以空串兜底），空值再套作者默认值：thumb 空 → `https://tc.uddz.cc/udid/xuuz.jpg`，url 空 → `https://tc.uddz.cc/xuuz/`，title 空 → 「个人名片」，desc 空 → 「描述内容」（**作者的推广卡：图床与主页都在其站点 tc.uddz.cc**）；
- 切全局队列调 `_internalSend…`（coverURL=thumb、linkURL=url），其返回的错误在主队列弹失败 toast。

`AutoMessageManager` 版 `sendLinkCardToConversation:` @0xa73b0 逻辑同构，只是改读 `xuuz_*` 单卡槽位（同一套作者默认值），如前述未找到调用点。

## 卡片数据与存储（问题 5）

NSUserDefaults（standardUserDefaults）键一览（全部实证）：

| Key | 内容 | 读/写方 |
|---|---|---|
| `WX_Card_List` | 卡片字典数组 | WXCardListViewController.loadCards @0xffca0 读（mutableCopy 进 `_cards`）；WXCardEditViewController.save @0xfe96c 写回并 post 通知 `WXCardListRefresh` |
| `WX_Selected_Card_Index` | 当前选中卡下标（NSInteger） | 列表 didSelect 写；VoiceEomjiSend.sendLinkCard 读 |
| `xuuz_title / xuuz_desc / xuuz_link_url / xuuz_cover_url / xuuz_keywords` | 单卡槽位（当前卡的字段副本） | 列表 didSelectRow @0x1024a0 写入；AutoMessageManager.sendLinkCard、`#卡片` hook、CustomMenuView 菜单 @0xb48b4 读取 |
| `AutoReplyKeywords / AutoReplyReplies / AutoReplyCardKeywords` | 自动回复三平行数组（关键词/文本回复/卡片关键词） | AutoReplyEditor.saveEntriesToUserDefaults @0xaca2c 写；TIMXOMessage hook 引擎读 |
| `AutoReplyEnabled` | 自动回复总开关（BOOL） | 引擎读 |
| `xuuz_hookim_enable` | `#卡片` 口令开关（BOOL） | sendMessage: hook 读 |

单条卡片字典（WXCardEditViewController.save 组装，键实证）：

```
{ title: 标题, desc: 描述, url: 跳转链接, thumb: 缩略图URL, keywords: 关键词 }
```

编辑页（WXCardEditViewController @0x10305d0）字段标签实证：标题 / 描述 / 链接地址（跳转URL）/ 缩略图URL / **关键词**（`keywordsField`）；保存后卡片进 `WX_Card_List`。

列表页（WXCardListViewController @0x1030620）行为实证：
- 点选一行 = 「设为当前卡」：把该卡五字段抄进 `xuuz_*` 槽位 + 写 `WX_Selected_Card_Index` + synchronize + reloadData（@0x1024a0）；列表 cell 显示当前选中标记与 keywords（cellForRowAtIndexPath: @0x100f8c）；
- 右上菜单（showAddMenu: @0x100c3c）：新增卡片 / 备份卡片 / 恢复卡片；
- 备份（backupCards @0x102ed4）：`WX_Card_List` → NSJSONSerialization（options=1，美化）→ 写 `NSTemporaryDirectory()/DY_Card_Backup.json` → UIActivityViewController 分享该文件；
- 恢复（restoreCards @0x103134）：UIDocumentPicker（UTType `json`，asCopy 打开）→ `documentPicker:didPickDocumentsAtURLs:` @0x103254 解析 JSON 写回 `WX_Card_List`（并触及 `WX_Selected_Card_Index`）。

## 复刻难度评估（问题 7）

**难度：低—中。** 构造本身极简（一个 model + 一个字典 + 一次发送调用），不需要 hook 抖音任何方法即可直发；难的是把抖音 IM 类名/方法名在目标版本上核准，以及确认服务端仍接受 type=26 的自组字典。

自研实现需要用到的抖音内部接口（全部按名动态调用即可，无需头文件）：
- `IESIMSendMessageModel`：`initWithConversationId:`、`setMessageType:`（卡片 26 / 文本 7）、`setMessageContentDict:`；
- `IESIMMessageSender`：`asyncSendMessage:completion:`（首选）、`sendMessage:`（兜底）；
- `TIMXSDKInstancesManager`：`iesim_TIMXSDKInstance`（判 SDK 就绪/取自身 ID）；
- 会话 ID 来源：消息对象的 `belongingConversationIdentifier`，或聊天页呈现时捕获；
- （口令路径才需要）hook `-[IESIMMessageSender sendMessage:]` 做原地改造 + `AWEIMTextMessage` / `AWEIMShareH5Message`（`initWithContentDict:`、`messageType`、`messageContentDict`）两个类名。

必须真机探针验证的点：
1. 目标抖音版本里 `IESIMSendMessageModel / IESIMMessageSender / TIMXSDKInstancesManager` 类名与上述方法签名是否仍在（作者代码里大量兜底与动态查选择子，说明他已在吃版本漂移的亏）；
2. type=26 是否仍是 H5 分享卡类型、`AWEIMShareH5Message initWithContentDict:` 产出的 messageType 实际值（本报告中 26 与之互证为推断链的一环，需实测闭环）；
3. 内容字典键集：5 键版（含驼峰 `coverURL`）与 4 键版哪个在当前版本渲染正常——作者两份代码不一致，疑似某版本改过键名，自研应双写保命（即照 5 键版）；
4. 服务端风控：自组 H5 卡（尤其外链域名非抖音合作域名时）是否被吞/被折叠/触发风控——作者默认卡用自家域名在真实分发，推断线上可达，但新域名需小号实测；
5. `initWithConversationId:` 接受的是会话 ID 字符串（本报告实证其用法），自研时勿传 TIMXOConversation 对象（VoicePackageViewController 的 `_currentConversation` 是 `TIMXOConversation` 类型，与本路径传字符串不同，勿混淆）。

## 附：关键地址速查

| 内容 | 地址 |
|---|---|
| ctor（TIMXO/SDK 捕获与自动回复 hook 注册） | 0xa050 |
| hook `-[TIMXOMessage initWithRootObject:model:]`（自动回复引擎） | IMP 0xa398 |
| hook `+[TIMXSDKInstancesManager iesim_TIMXSDKInstance]` | IMP 0xa124 |
| ctor（`#卡片` 口令 hook 注册） | 0x10f94 |
| hook `-[IESIMMessageSender sendMessage:]`（`#卡片` 改造） | IMP 0x14b60 |
| hook `-[AWEIMNewMessageDataController sendMessage:mentionUsers:customToast:enterFrom:]` | IMP 0x14a28 |
| 手动发送当前卡（C 函数，读全局会话 ID） | 0x8f40 |
| 全局：当前会话 ID / 本机用户 ID | 0x1508010 / 0x1508030 |
| VoiceEomjiSend sendCard / sendLinkCard / _internalSend | 0xe3c44 / 0xe3f24 / 0xe44f0 |
| AutoMessageManager sendLinkCard / _internalSend | 0xa73b0 / 0xa787c |
| WXCardEditViewController.save / WXCardListViewController.loadCards | 0xfe96c / 0xffca0 |
| AutoReplyEditor.openCardConfig / saveEntriesToUserDefaults | 0xacd00 / 0xaca2c |
