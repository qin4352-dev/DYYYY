# DouyinHelper.dylib「评论字体颜色」功能反汇编分析报告

- 目标：`DouyinHelper.dylib`（Mach-O arm64，18,192,944 字节，sha256 79d6cf78…），作者 Xuu 的抖音注入 tweak
- 分析方式：手工解析 ObjC 元数据（LIEF 本构建无 ObjC 元数据支持）+ capstone 反汇编 + 字符串交叉引用
- 原始 dump 同目录：`objc_classes_methods.txt`、`strings_comment_color.txt`、`xrefs_keys.txt`、`xrefs_extraInfo.txt`、`hook_table.txt`、`disasm_hookIMP_setExtraInfo.txt`、`disasm_previewForPattern.txt`、`disasm_sendpath.txt`

## 结论先行

**颜色不是画在本地界面上的，而是伪造抖音原生的「文本分段」（text extra）数据、随评论一起提交到服务器，所以任何人用任何客户端看这条评论都是带颜色的。**

具体机制：tweak 用 `MSHookMessageEx` 挂钩了抖音评论输入内容模型的方法 `-[AWECommentInputContentModel setExtraInfo:]`。用户点发送时，抖音会把这条评论的附带分段数组（extraInfo，本来装 @提及、话题等 `AWETextExtra` 对象）写进这个模型。tweak 在写入瞬间拦截：若功能开关开启，就按选定样式把整段评论文字切成若干区间，为每个区间造一个 `AWETextExtra` 对象，其 `type` 字段填 **21（红）** 或 **1（蓝）**、`subtype` 填 0、`start`/`end` 填区间下标，再用这批假分段替换掉原数组里全部 `AWETextExtra`（非 AWETextExtra 的原始元素保留），最后调用原实现。这个数组会随评论发布请求序列化上行（评论的 text_extra），抖音各端在渲染评论时按 extra 的 type 给对应区间套用该类型自带的主题高亮色——于是别人看到的就是红/蓝文字。这也解释了作者自己的说明「浅色模式显示红色、深色模式显示黄色」：颜色由 type 决定、随抖音主题变，不是写死的 RGB。

## 证据

### 挂钩点（构造函数直接注册，全二进制唯一一处）

构造函数 `__ctor_8` @ **0xf7e8**（`__init_offsets` 第 8 项），全文仅 8 条指令：

```
0xf7f0: adrp/add x0 -> cstr 'AWECommentInputContentModel'  @0xcbee25
0xf7f8: objc_getClass("AWECommentInputContentModel")
0xf800: ldr x1 <- selref @0x102bed0  'setExtraInfo:'
0xf804: adr x2 -> 0xf81c             (替换 IMP)
0xf80c: x3 = 0x103b980               (原 IMP 保存槽, __DATA)
0xf818: b MSHookMessageEx
```

交叉引用验证：`setExtraInfo:` 的 selref 与 `AWECommentInputContentModel` 字符串在整个 `__text` 里**只**被这处引用——没有第二个（显示侧）挂钩点。

### 替换 IMP 与核心例程地址

| 内容 | 地址 |
|---|---|
| 替换 IMP（hook 本体） | **0xf81c** – 0xfe2c |
| 造分段的 block invoke（addSegment） | **0xfed4** – 0x10058 |
| 原 IMP 保存槽 | 0x103b980 |
| 设置页类 `XUUZCommentFontColorSettingsVC` | 类结构 @0x1030670 |
| 设置页预览方法 `-previewForPattern:text:` | IMP @0x103ba0（与 hook 共用同一套样式编号） |
| hook 内样式跳转表 | @0xa6fb48（基址 0xfa84） |
| 预览样式跳转表 | @0xa70038（基址 0x103d3c） |

### 设置存储（NSUserDefaults，standardUserDefaults）

| Key | CFString 地址 | 类型 | 读写方 |
|---|---|---|---|
| `XUUZRedTextEnabled` | @0xff0d48（cstr @0xcbee74） | BOOL | 设置页 `setBool:forKey:`；hook 内 `objectForKey:`+`boolForKey:` |
| `XUUZRedTextPattern` | @0xff0d68（cstr @0xcbee87） | NSInteger 0–11 | 设置页 `setInteger:forKey:`；hook 内 `objectForKey:`+`integerForKey:` |

语义（hook 与设置页 getter 完全一致）：Enabled 的 key **不存在或为 NO → 功能关**（原样透传）；Pattern 的 key 不存在 → 0；读到的值 **> 11 → 按 0 处理**。

### 功能文案（UTF-16 字符串，作者自述，与反汇编互证）

- 样式表 @0xd9fd22：「0 全红｜1 全蓝｜2 红蓝交替｜3 蓝红交替｜4 红红蓝蓝｜5 蓝蓝红红｜6 首尾红｜7 首尾蓝｜8 只首字红｜9 只首字蓝｜10 只尾红｜11 只尾蓝（默认 0；非 0~11 的值按 0 处理）」
- 说明 @0xd9fc8e：「启用后 你评论的文字将显示为红色或蓝色 默认为全红色 ⚠️浅色模式下是显示为红色 深色模式下是显示为黄色 ⚠️下方可调整颜色样式」
- 样式名 @0xda3be0 起：全红/全蓝/红蓝交替/蓝红交替/红红蓝蓝/蓝蓝红红/首尾红/首尾蓝/只首字红/只首字蓝/只尾字红/只尾字蓝

## 分段 / 数据格式细节

每个分段是一个抖音原生类 **`AWETextExtra`** 对象（hook 用 `NSClassFromString(@"AWETextExtra")` @0xff0ca8 取得类，`alloc/init` 后用 **KVC `setValue:forKey:`** 赋值——不依赖任何抖音头文件声明）：

| KVC key | CFString 地址 | 值 | 类型 |
|---|---|---|---|
| `type` | @0xff0cc8 | **21 = 红，1 = 蓝** | NSNumber（numberWithLongLong:） |
| `subtype` | @0xff0ce8 | 恒 0 | NSNumber（numberWithInt:） |
| `start` | @0xff0d08 | 区间起始下标 | NSNumber（numberWithLongLong:） |
| `end` | @0xff0d28 | 区间结束下标 | NSNumber（numberWithLongLong:） |

- 下标是评论文本（`-[AWECommentInputContentModel content]`）的 NSString 下标（UTF-16 码元计，与 `-[NSString length]` 一致）。
- 区间在造段时钳制：`start = max(start, 0)`，`end = min(文本长度, end)`，`start >= end` 的空区间跳过不造。
- 没有颜色字符串、没有十六进制色值——颜色完全由 `type` 数值决定。
- 21↔红、1↔蓝的配对由设置页预览方法实证：`previewForPattern:text:` 用 `[UIColor systemRedColor]` / `systemBlueColor` + `NSForegroundColorAttributeName` 逐字上色，其样式分支顺序与 hook 的 type 分支（0x15 / 0x1）逐项对应（见下表）。

### 样式 → 分段映射（n = 文本长度，下标区间左闭右开）

| # | 样式 | hook 生成的分段（type, [start, end)） |
|---|---|---|
| 0 | 全红 | (21, [0, n)) |
| 1 | 全蓝 | (1, [0, n)) |
| 2 | 红蓝交替 | 每字一段：偶数位 (21, [i, i+1))，奇数位 (1, [i, i+1)) |
| 3 | 蓝红交替 | 每字一段：偶数位 (1, …)，奇数位 (21, …) |
| 4 | 红红蓝蓝 | 见下方注记（成对发出，步进 2） |
| 5 | 蓝蓝红红 | 同上，红蓝互换 |
| 6 | 首尾红 | 每字一段：首、尾字 (21, …)，中间 (1, …) |
| 7 | 首尾蓝 | 每字一段：首、尾字 (1, …)，中间 (21, …) |
| 8 | 只首字红 | (21, [0, 1)) + (1, [1, n)) |
| 9 | 只首字蓝 | (1, [0, 1)) + (21, [1, n)) |
| 10 | 只尾字红 | (1, [0, n-1)) + (21, [n-1, n)) |
| 11 | 只尾字蓝 | (21, [0, n-1)) + (1, [n-1, n)) |

**注记（实证的作者 bug）**：样式 4/5 在 hook 里的成对循环步进是 2 而不是 4——每轮发出 [i, i+2) 与 [i+2, i+4) 两段后，下一轮从 i+2 开始，于是内侧区间会被重复发出一遍另一色的段。例如 n=6、样式 4 实际生成：`(21,[0,2)) (1,[2,4)) (21,[2,4)) (1,[4,6)) (21,[4,6))`（含重叠段）。而设置页预览的成对循环步进是 4（干净的 [0,2) 红、[2,4) 蓝……）。即**预览与真实发送在样式 4/5 上并不一致**；重叠段在抖音渲染端如何裁决（先到先得/后到覆盖）无法从本二进制确定。

### 合并规则（对原 extraInfo 的处理）

1. 先按样式生成新分段数组 `NSMutableArray`；
2. 快速枚举**原始** extraInfo（入参为 nil 时按空数组）：元素**不是** `AWETextExtra` 实例的，追加到新数组末尾；是 `AWETextExtra` 的**一律丢弃**（包括用户真实 @提及 产生的分段——这是该实现的已知副作用）；
3. 以合并后的数组调用原 `setExtraInfo:`。

### 透传（不注入）条件

以下任一成立，hook 直接以原入参调用原实现：Enabled key 不存在或为 NO；`content` 为空（length 0）；`AWETextExtra` 类不存在。

## 核心例程伪代码

```objc
// 注册（__ctor_8 @0xf7e8）
%ctor {
    MSHookMessageEx(objc_getClass("AWECommentInputContentModel"),
                    @selector(setExtraInfo:), hook_setExtraInfo, &orig_setExtraInfo);
}

// hook 本体（IMP @0xf81c）
static void hook_setExtraInfo(AWECommentInputContentModel *self, SEL _cmd, NSArray *extraInfo) {
    NSUserDefaults *d = [NSUserDefaults standardUserDefaults];
    if ([d objectForKey:@"XUUZRedTextEnabled"] == nil || ![d boolForKey:@"XUUZRedTextEnabled"])
        return orig_setExtraInfo(self, _cmd, extraInfo);
    NSString *content = [self content];
    if (content.length == 0) return orig_setExtraInfo(self, _cmd, extraInfo);
    Class extraCls = NSClassFromString(@"AWETextExtra");
    if (!extraCls) return orig_setExtraInfo(self, _cmd, extraInfo);

    NSInteger pattern = 0;
    if ([d objectForKey:@"XUUZRedTextPattern"] != nil) {
        pattern = [d integerForKey:@"XUUZRedTextPattern"];
        if (pattern > 11) pattern = 0;
    }

    NSMutableArray *out = [NSMutableArray array];
    // addSegment block（invoke @0xfed4）：钳制区间后造段
    void (^addSegment)(NSInteger type, NSInteger start, NSInteger end) = ^(NSInteger type, NSInteger s, NSInteger e) {
        s = MAX(s, 0); e = MIN((NSInteger)content.length, e);
        if (s >= e) return;
        AWETextExtra *ex = [[extraCls alloc] init];
        [ex setValue:@(type) forKey:@"type"];      // 21 红 / 1 蓝
        [ex setValue:@(0)    forKey:@"subtype"];
        [ex setValue:@(s)    forKey:@"start"];
        [ex setValue:@(e)    forKey:@"end"];
        [out addObject:ex];
    };

    switch (pattern) {           // 12 个分支，映射见上表
        case 0: addSegment(21, 0, n); break;
        case 1: addSegment(1, 0, n); break;
        case 2: for (i in 0..<n) addSegment(i % 2 == 0 ? 21 : 1, i, i + 1); break;
        // …其余类推（4/5 为步进 2 的成对循环，含重叠段，见注记）
    }

    for (id item in extraInfo ?: @[])
        if (![item isKindOfClass:extraCls]) [out addObject:item];
    orig_setExtraInfo(self, _cmd, out);
}
```

## 自研复刻要点

在我们自己的 Logos tweak 里实现同样功能，照抄这条路线即可，不需要碰网络层、不需要改评论请求：

1. **挂钩点**：`%hook AWECommentInputContentModel` 的 `-setExtraInfo:`（该类是抖音评论输入内容模型；`content` 属性即评论文本）。在 `%new`/替换实现里先读开关，再决定透传还是改写，最后 `%orig(合并后的数组)`。
2. **造段**：`NSClassFromString(@"AWETextExtra")` 取类（不要硬链接抖音类），`alloc/init` 后 **KVC 赋 4 个键**：`type`（21 红 / 1 蓝）、`subtype`（0）、`start`、`end`（均为 NSNumber）。区间用文本的 UTF-16 下标，并做钳制与空区间跳过。
3. **样式引擎**：一个 `(type, start, end)` 的分段生成函数 + 12 路 switch（映射表见上）。逐字样式（2/3/6/7）每字一段；整段样式（0/1/8/9/10/11）只需 1–2 段。
4. **合并策略**：生成的颜色段在前，原 extraInfo 中非 `AWETextExtra` 的元素保留追加；是否丢弃原 `AWETextExtra`（原作者丢弃，会误伤真实 @提及）可自行决定——更稳妥的做法是保留原分段、仅追加不重叠的颜色段，但要注意抖音对重叠区间的渲染裁决未知，需实机验证。
5. **设置存储**：`NSUserDefaults` 两个键（开关 BOOL + 样式 NSInteger），设置页与 hook 共用同一读取逻辑（缺省：关 / 样式 0 / 越界归 0）。设置页预览可用 `NSAttributedString` + `NSForegroundColorAttributeName`（systemRed/systemBlue）本地渲染，与发送无关。
6. **成对样式（4/5）注意**：复刻时用步进 4 的干净实现（与预览一致）；原作者 hook 的步进 2 重叠发段是 bug，不必照抄，除非要 1:1 复现其线上效果（需先实机确认抖音对重叠段的裁决）。
7. **验证方法**：发一条带颜色段的评论后，用无插件的干净账号查看——颜色在即证明分段已上行（本功能的用户侧验收标准）。

## 未确定项与置信度

**实证（反汇编直接读出，置信度高）**：
- 挂钩类/方法/注册方式、替换 IMP 与造段 block 的全部逻辑、4 个 KVC 键与 type 值 21/1、subtype 恒 0、区间钳制规则、12 种样式的分支结构、合并与透传规则、两个 defaults key 及缺省语义、设置页与 hook 的对应关系、样式 4/5 的重叠发段现象。

**推断（合理但未在本二进制内闭环）**：
- `type` 21 与 1 在抖音 `AWETextExtra` 枚举中的官方名称：本二进制只给出数值与红/蓝配对。type=1 疑为 @提及类分段的原生类型，21 的原生语义需查 Aweme 侧头文件才能定名；「深色模式显示黄色」是作者文案，其渲染细节属抖音客户端行为，未独立验证。
- extraInfo 数组随评论请求上行的具体字段名（text_extra）与序列化格式在抖音 App 侧，不在本 dylib 内；「别人能看到」的结论由机制（改的是发布前的内容模型数据，而非本地视图）+ 用户实测（他人账号可见红字）共同支撑。
- 抖音渲染端对重叠分段的裁决规则未知（影响样式 4/5 的实际观感）。
- `setExtraInfo:` 在抖音内部的全部调用时机未枚举；hook 在该方法每次被调用时都会改写，若抖音在草稿恢复等场景也调用它，可能产生额外注入（未见防护代码）。

**已排除**：直播间弹幕发送路径（`HTSLiveCommentHelper +sendMsgFromDefaults:` @0x114940，走 `IESLiveCommentContext` + `sendComment:commentContext:completion:`）是另一个「预设弹幕」功能，与视频评论颜色无关；DYYY 开源源码中只有弹幕改色、没有此功能，本功能为 Xuu 原创部分，与 DYYY 无对应代码。
