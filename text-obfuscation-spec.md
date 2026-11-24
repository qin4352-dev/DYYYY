# 抖音发评论「防过滤混淆」规格 v1

- 定位：我们自研抖音插件的发送侧功能，与「评论字体颜色」（见 `xuu-comment-color-analysis.md`）同一挂钩层、同一条管线。
- 一句话：**发送前把评论文字做不可见混淆——人眼看完全正常，但关键词精确匹配、复制搜索这类机器匹配抓不准。** 这是混淆，不是加密：原文不需要解码，渲染时零宽字符天然不可见。

## 原理与手段（分层，可独立开关）

### 第 1 层（主力）：零宽字符随机插入
- 在字符之间插入零宽字符，字符池只用三个：`U+200B`（零宽空格）、`U+200C`（零宽非连接符）、`U+FEFF`（零宽不换行空格）。**绝不用 `U+200D`（零宽连接符）**——它会真实参与 emoji/文字的连接渲染。
- 效果：任何长度 ≥2 的关键词，其连续字符串被打断，精确子串匹配失效；别人复制你的评论去搜，搜到的文本也带着零宽字符，对不上原文。
- 密度三档（设置项，见下）：
  - 低：每个字缝约 1/3 概率插入
  - 中：约 2/3 概率
  - 高：每个字缝必插（最强，但有效字数近乎减半，见局限）
- 随机两维：插不插随机、用哪个零宽字符随机——避免形成固定节律的机器特征。

### 第 2 层（可选，默认关）：形近字替换
- 对用户自维护的敏感词表做形近/异体字替换（如 微→薇、信→伩、钱→銭 类）。需要词表驱动，只替换词表命中的字，未命中不动。
- 本规格只留接口与数据格式（词表 = 字典 JSON：原字→替换字），词表内容由使用者自备，不内置。

### 不做的
- 不用真密文（base64/乱码）：公开评论发乱码会被当垃圾内容，且需要显示侧解码，闭环成本高——用户已明确选的是防过滤混淆，不是密文。
- 不做繁简全量转换：对中文关键词表有一定干扰作用，但改变阅读习惯且收益低，不值当。

## 安全规则（实现时必须守）
1. **只在字素簇（grapheme cluster）边界插入**：用 `NSStringEnumerationByComposedCharacterSequences` 枚举，绝不插进代理对（emoji 高低位）中间、绝不把组合附加符号（变音符、变体选择符 VS16、肤色修饰符）与基字拆开、绝不拆国旗的区域指示符对。插错位置会让 emoji 显示成两个方块或问号。
2. **文本末尾不插入**：最后一个字缝之后不加零宽字符（无意义且易被注意）。
3. **空文本/单字不处理**：length 0 直接透传；单字没有字缝，天然不插。
4. **变换必须幂等可检测**：若文本已含池中零宽字符（比如粘贴来的），先剥离再按密度重插，避免叠加膨胀。

## 与评论颜色功能的管线顺序（关键）
两个功能改的是同一个模型（`AWECommentInputContentModel`）的两个面：混淆改 `content` 本身，颜色改 `extraInfo` 分段数组。顺序与下标规则定死如下：

1. **先混淆，后分段**：颜色分段的 `start`/`end` 是最终文本（混淆后）的 UTF-16 下标。必须在文本定稿后才造段。
2. **样式步进按“可见字”计，不按最终下标计**：造段时枚举最终文本的字素簇，纯零宽簇不推进样式步进（不占红/蓝交替的位置），但它的下标区间要并入相邻分段的范围内（分段是连续区间，零宽字符落在哪个区间就归哪个区间）。
   - 反例（禁止）：对混淆后的字符串逐 UTF-16 单位交替上色——零宽字符会吃掉一半颜色位，可见字会变成清一色。
3. 挂钩点：
   - 混淆：`-[AWECommentInputContentModel setContent:]`，在 `%orig` 前把入参换成混淆后的字符串（setter 名按 ObjC 惯例推定，**待探针验证**，见文末）。
   - 颜色：沿用已实证的 `setExtraInfo:` 改写（见分析报告）。两者的实际调用先后由抖音内部决定，但因为颜色造段读的是 `self.content`（届时已是混淆后文本），且按上面第 2 条造段，结果自洽。
4. 两功能开关独立：只开混淆不开颜色时，不动 extraInfo；只开颜色时文本原样。

## 设置项（按 Xuu 面板模板：一功能一页、开关带后果句）
- 总开关「启用发送混淆」——后果句：关闭后评论按原文发送，不做任何处理。
- 密度单选：低 / 中 / 高（行内注明高密度下可发字数约减半）。
- 第 2 层开关「形近字替换」+ 词表编辑入口（默认关；词表为空时开启也无效果，行内明示）。
- 存储 key（本项目命名空间，实现时可改名）：`DYObfEnabled`（BOOL，缺省关）、`DYObfDensity`（0/1/2 = 低/中/高，缺省 0 低）、`DYObfHomoglyph`（BOOL，缺省关）、`DYObfHomoglyphMap`（JSON 字符串）。

## 核心代码（参考实现，可直接进 Logos tweak）

```objc
// ---- 第 1 层：零宽插入 ----
static BOOL DYIsZeroWidthCluster(NSString *s) {
    // 纯零宽簇判定：组成字符全在池中
    for (NSUInteger i = 0; i < s.length; i++) {
        unichar c = [s characterAtIndex:i];
        if (c != 0x200B && c != 0x200C && c != 0xFEFF) return NO;
    }
    return s.length > 0;
}

static NSString *DYStripZeroWidth(NSString *text) {
    NSMutableString *out = [NSMutableString stringWithCapacity:text.length];
    for (NSUInteger i = 0; i < text.length; i++) {
        unichar c = [text characterAtIndex:i];
        if (c == 0x200B || c == 0x200C || c == 0xFEFF) continue;
        [out appendString:[NSString stringWithCharacters:&c length:1]];
    }
    return out;
}

// density: 0=低(1/3) 1=中(2/3) 2=高(必插)
NSString *DYObfuscatedString(NSString *text, NSInteger density) {
    if (text.length == 0) return text;
    NSString *clean = DYStripZeroWidth(text);            // 幂等：先剥旧的
    if (clean.length <= 1) return clean;
    static NSArray<NSString *> *pool;
    static dispatch_once_t once;
    dispatch_once(&once, ^{ pool = @[@"\u200B", @"\u200C", @"\uFEFF"]; });

    NSMutableArray<NSString *> *clusters = [NSMutableArray array];
    [clean enumerateSubstringsInRange:NSMakeRange(0, clean.length)
        options:NSStringEnumerationByComposedCharacterSequences
        usingBlock:^(NSString *sub, NSRange r, NSRange er, BOOL *stop) {
            [clusters addObject:sub];
        }];
    if (clusters.count <= 1) return clean;

    NSMutableString *out = [NSMutableString stringWithCapacity:clean.length * 2];
    for (NSUInteger i = 0; i < clusters.count; i++) {
        [out appendString:clusters[i]];
        if (i + 1 == clusters.count) break;              // 末尾不插
        BOOL insert;
        switch (density) {
            case 0:  insert = (arc4random_uniform(3) == 0); break;
            case 1:  insert = (arc4random_uniform(3) != 0); break;
            default: insert = YES; break;
        }
        if (insert) [out appendString:pool[arc4random_uniform(3)]];
    }
    return out;
}
```

颜色造段侧的配合（伪代码，完整造段逻辑见分析报告）：枚举最终文本字素簇时维护 `visibleIndex`（纯零宽簇不递增）与每个簇的 UTF-16 起止；样式函数按 `visibleIndex` 决定 type（21/1），生成分段时把区间端点映射回最终文本下标，零宽簇的区间自然并入覆盖它的那一段。

第 2 层形近字替换：在第 1 层之前执行——按词表对 clean 文本做逐字替换（替换字与原字同为单字时不影响字素簇结构），然后再走零宽插入。

## 局限与风险（写进面板文案，不藏着）
- **不是万能的**：平台若在入库/审核前做归一化（剥零宽字符再匹配），本层即失效。它对付的是未归一化的精确匹配、以及其他用户侧的复制搜索/关键词举报，属抬高门槛，不是保证。
- **字数成本**：抖音评论有长度上限且按字符计，零宽字符也占位——中密度下可发正文约剩六成，高密度约对半。面板在高密度行内明示。
- **账号纪律**：沿用既有规矩，只在小号启用、只测可抛弃账号。
- 形近字替换过度会伤可读性，也可能被人工审核一眼识破意图，词表宁短勿滥。

## 待探针验证（正式开工时先做，别猜）
1. `AWECommentInputContentModel` 的文本 setter 的确切名字与调用时机（是否发布前必经；草稿恢复是否也走，会不会重复混淆——本设计有幂等剥离，可重复进入不膨胀，但要实测确认）。
2. 发布请求最终读取的是模型的 `content` 属性还是别处的副本（决定混淆挂钩点是否充分）。
3. 混淆文本 + 颜色分段同发时，抖音渲染端对含零宽字符区间的着色表现（应无可见差异，实测为准）。
4. 抖音评论长度上限的计数口径（按 UTF-16 单位还是字素簇），决定密度档的实际可用长度。
