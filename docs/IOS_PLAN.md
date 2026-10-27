# 饮食日志 iOS 版：开发计划

这份文档是给自己和 VS Code 里的 Claude Code 用的：先读完这份，再按第 7 节的里程碑一步一步做，每做完一步都要能编译、测试通过。

网页版（仓库根目录的 `index.html`）是功能和数据的参照，`seed/` 里是现在的食物库和目标。

---

## 0. 目标和范围

做一个**自己用的**原生 iOS App，功能和网页版对齐，不需要服务器、不需要自己写登录。

做：

- 点选记录：早餐 / 午餐 / 晚餐 / 加餐，点食物加一份，格子上有 − / 数量 / +
- 营养成分表卡片：热量、蛋白、碳水、脂肪，已摄入 / 目标 / 占比，各餐蛋白分布；热量可在 kcal / kJ 之间切换
- 打开时按本地时间自动选中早餐 / 午餐 / 晚餐 / 加餐
- 记录里每一条按份数显示总量（如「2 个 · 140 kcal · 12g 蛋白」）
- 训练日 / 休息日两套目标，默认按星期几判断，单天可手动切换；「改目标」页
- 食物按近 7 天记录次数排序，次数相同按名称排（中文按拼音，英文按字母）
- 记录可改份数（支持小数）、删除、清空一餐、存为套餐，所有操作可撤销
- 一次性记录、新增 / 编辑 / 删除食物，热量可以为 0
- 中英双语界面；食物有中英文名，当前语言名字必填，另一种留空就自动翻译
- 近 14 天：每天热量对比目标，近 7 天平均热量和蛋白

不做（至少第一版）：账号系统、服务器、多人共享、上架 App Store。

---

## 1. 需要你自己动手的部分（Claude Code 做不了）

这些步骤要你在 Mac 的图形界面或 Apple 网站上操作：

1. **安装 Xcode**（Mac App Store），装好后打开一次，让它装完组件。然后在终端运行：
   ```bash
   sudo xcode-select -s /Applications/Xcode.app
   xcodebuild -version
   ```
2. **Xcode 登录 Apple ID**：Xcode › Settings › Accounts › 添加你的 Apple ID。
3. **决定要不要买 Apple Developer Program（99 美元/年）**：
   - 免费账号：能装到自己 iPhone，但签名 **7 天过期**，过期后要连 Mac 重新装；**不能用 iCloud / CloudKit 同步**。
   - 付费账号：签名一年有效，能用 TestFlight（手机上直接装更新），能开 iCloud 同步。
   - 建议：先用免费账号把 M0–M5 做完，确定自己天天在用，再买。
   - 各能力对哪种账号开放，以 Apple 的[能力对照表](https://developer.apple.com/help/account/reference/supported-capabilities-ios/)为准。
4. **iPhone 打开开发者模式**：设置 › 隐私与安全性 › 开发者模式，用数据线连 Mac，在手机上点「信任此电脑」。
5. **第一次在真机上运行**：在 Xcode 里打开项目，Signing & Capabilities 选你的 Team，点运行。之后可以回到 VS Code 用命令行构建。
6. **（M6 才需要）开 iCloud**：付费账号下，在 Xcode 的 Signing & Capabilities 加 iCloud → 勾 CloudKit → 新建容器 `iCloud.com.jiachengzhao.dietlog`。上线前在 CloudKit Console 把开发环境的 Schema 部署到生产环境。

---

## 2. VS Code + Claude Code 的工作方式

iOS 项目可以全程在 VS Code 里写代码，构建和运行交给命令行，只有签名和开 capability 这类事要回 Xcode 点一下。

**工具**（Homebrew 安装）：

```bash
brew install xcodegen xcbeautify
```

- **XcodeGen**：用 `ios/project.yml` 描述项目，自动生成 `.xcodeproj`。这样 Claude Code 只改 YAML 和 Swift 文件，不用手改容易出错的 `project.pbxproj`。生成的 `.xcodeproj` 不进 Git。
- **xcbeautify**：让 `xcodebuild` 的输出好读。
- **VS Code 扩展**：Swift（官方，提供补全和跳转）；可选 SweetPad（在 VS Code 里一键构建并跑模拟器）。

**目录结构**（建在本仓库的 `ios/` 下）：

```
ios/
  project.yml          # XcodeGen 项目定义
  Makefile             # gen / build / test / run
  CLAUDE.md            # 给 Claude Code 的项目约定（M0 生成）
  DietLog/
    App/               # App 入口、根视图
    Models/            # SwiftData 模型
    Logic/             # 纯逻辑：合计、日类型、排序、日期工具（可单测）
    Views/             # Today、Picker、Log、Targets、Foods、History、组件
    Services/          # 种子导入、翻译、（以后）HealthKit
    Resources/         # Localizable.xcstrings、seed JSON、Assets
  DietLogTests/
```

**常用命令**（写进 Makefile）：

```bash
cd ios && xcodegen generate
xcrun simctl list devices available            # 看有哪些模拟器
xcodebuild -project DietLog.xcodeproj -scheme DietLog \
  -destination 'platform=iOS Simulator,name=iPhone 16' build | xcbeautify
xcodebuild -project DietLog.xcodeproj -scheme DietLog \
  -destination 'platform=iOS Simulator,name=iPhone 16' test | xcbeautify
```

模拟器名字以 `simctl list` 的结果为准。

**和 Claude Code 协作的节奏**：一次只给一个里程碑；让它每步结束跑 `make build` 和 `make test`；涉及 Xcode 图形界面的步骤，让它停下来告诉你点哪里。

---

## 3. 技术选型

| 部分 | 选择 | 说明 |
| --- | --- | --- |
| UI | SwiftUI | 最低支持 **iOS 18**（SwiftData 稳定版 + 程序化翻译 API） |
| 本地数据 | SwiftData | 不需要服务器；以后可直接开 iCloud 同步 |
| 同步 | SwiftData 内置的 CloudKit 同步（M6） | 自动用你的 Apple ID，不用写登录；需要付费账号 |
| 图表 | Swift Charts | 近 14 天柱状图 + 目标线 |
| 翻译 | Apple Translation 框架 | 在设备上翻译，免费，不用联网调用 Claude |
| 健康数据（可选） | HealthKit | 把每天的热量、蛋白、碳水、脂肪写进「健康」App |
| 小组件（可选） | WidgetKit | 主屏显示今天还剩多少 kcal |
| 第三方依赖 | 无 | 全用系统框架 |

**SwiftData + CloudKit 的硬性约束**（从 M1 起就按这个写，免得 M6 返工）：

- 每个属性都要有默认值，或者是可选类型
- 不能用 `@Attribute(.unique)`，唯一性要在代码里自己保证
- 所有关系都必须是可选的
- 同步上线后，Schema 只能加字段，不能删字段或改字段类型

---

## 4. 数据模型（对应网页版）

和网页版的主要区别：网页版是「一天一个文档，里面一个 entries 数组」；iOS 版把每条记录拆成一行 `Entry`，按日期查询、算近 7 天排行都更方便。

```swift
@Model final class Food {
    var key: String = ""            // 网页版的 id，如 "egg"；新建的用 UUID 字符串
    var nameEn: String = ""
    var nameZh: String = ""
    var unitEn: String = ""
    var unitZh: String = ""
    var kcal: Double = 0            // 每份
    var protein: Double = 0
    var carbs: Double = 0
    var fat: Double = 0
    var category: String = "Other"  // 英文分类键，界面上再翻译
    var sort: Int = 999
    var createdAt: Date = Date()
}

@Model final class Entry {          // 一条记录
    var date: String = ""           // 本地日期 "YYYY-MM-DD"
    var meal: String = "b"          // b / l / d / s
    var foodKey: String?            // 一次性记录为 nil
    var nameEn: String = ""         // 记录当时的快照，改食物不影响历史
    var nameZh: String = ""
    var unitEn: String = ""
    var unitZh: String = ""
    var kcal: Double = 0            // 每份
    var protein: Double = 0
    var carbs: Double = 0
    var fat: Double = 0
    var qty: Double = 1
    var createdAt: Date = Date()
}

@Model final class MealPreset {     // 套餐
    var name: String = ""
    var slot: String = "s"
    var sort: Int = 999
    @Relationship(deleteRule: .cascade, inverse: \PresetItem.preset)
    var items: [PresetItem]? = []
}

@Model final class PresetItem {
    var foodKey: String = ""
    var qty: Double = 1
    var preset: MealPreset?
}

@Model final class DayInfo {        // 只在手动切换训练/休息时才有
    var date: String = ""
    var dayType: String?            // "train" / "rest" / nil = 按星期几
}

@Model final class Targets {        // 单例
    var trainKcal: Double = 2200
    var trainProtein: Double = 145
    var trainCarbs: Double = 290
    var trainFat: Double = 50
    var restKcal: Double = 1900
    var restProtein: Double = 135
    var restCarbs: Double = 130
    var restFat: Double = 80
    var trainDays: [Int] = [0, 4]   // 0 = 周日；现在是周四、周日
    var updatedAt: Date = Date()
}
```

几个约定：

- `Entry` 用 `foodKey` 字符串关联食物，不用关系。删掉一个食物时，历史记录照样保留。
- `Targets` 和 `DayInfo` 的「唯一」靠代码保证。开同步后可能出现重复：`Targets` 取 `updatedAt` 最新的那条，`DayInfo` 同一天取最新的。
- 日期一律用本地时区的 `"YYYY-MM-DD"` 字符串，跟网页版一致，跨时区出差也不会把记录算到别的日子。

---

## 5. 迁移现有数据

1. 把 `seed/foods.json`、`seed/presets.json` 和 `seed/settings.json` 放进 App 的资源目录；首次启动时导入。按 `key` 去重，重复启动不会导入第二遍。
   - 食物字段对应：`name`→`nameEn`，`zh`→`nameZh`，`unit`→`unitEn`，`unit_zh`→`unitZh`，`p/c/f`→`protein/carbs/fat`，`cat`→`category`。
   - 套餐：现在有 2 个（海鲜拉面、咖喱海鲜饭），`items[].food` 对应 `Food.key`。导入成 `MealPreset` + `PresetItem`。
2. 历史饮食记录不在仓库里。需要的话，在 claude.ai 里让 Claude 把网页版数据库的 `days` 集合导出成 JSON，存为 `seed/days.json`，再写一个导入器：把每天的 `entries` 数组拆成一行行 `Entry`，`dayType` 不为空的写进 `DayInfo`。

---

## 6. 页面与逻辑（逐项对齐网页版）

**今日页**（主页面）

1. 顶部：‹ 日期 ›、「回到今天」、训练日 / 休息日切换、还剩多少热量和蛋白进度。
2. 营养成分表卡片：沿用网页版的美国营养标签样式（粗黑线、大号热量数字），右上角「✎ 改目标」。「能量」下面有 kcal | kJ 切换（见下方「热量单位」）。
3. 餐次分段控件：按设备本地时间（也就是用户所在时区）自动选。
   - 05:00 – 11:30：早餐
   - 11:30 – 16:30：午餐
   - 16:30 – 21:00：晚餐
   - 21:00 – 次日 05:00：加餐
   - 用户手动点了某一餐，就保持手动选择；App 从后台回到前台时，没手动选过才按当前时间重选（监听 `scenePhase` 变为 `.active`）。
   - 切回「今天」时重新按时间选，并清掉「手动选过」的标记。
4. 套餐：横向标签，点一下把整顿记到它自己的那一餐；管理模式下点两次删除。
5. 食物网格：
   - 分类标签和搜索框，搜索同时匹配中英文名和份量。
   - 每格显示名称、份量、「本周 N 次」、kcal 和蛋白，下面一行是 − 数量 +。
   - 点格子本身也是 +1。
6. 记录列表：
   - 按餐分组，每组显示小计；份数可以用 ±0.5 调，也可以直接输入小数。
   - 每条第二行显示**按份数算的总量**：份量 × 份数、热量 × 份数、蛋白 × 份数，比如水煮蛋 2 份显示「2 个 · 140 kcal · 12g 蛋白」。右边不再单独放一列总热量。
   - 每条有删除，每组有「清空这餐」「存为套餐」。
   - 每次操作后底部出现提示条，带「撤销」按钮。

**份量随份数缩放（`scaleUnit`）**：

- 份数为 1 时原样显示。
- 份量以数字开头（含 ¼ ½ ¾）时，把开头的数字乘以份数：「1 个」×2 →「2 个」，「200 ml」×2 →「400 ml」，「¼ lb 生重」×2 →「0.5 lb 生重」，「100 g dry」×0.9 →「90 g dry」。结果最多保留两位小数。
- 英文份量里紧跟数字的单词变复数：egg→eggs、slice→slices、bowl→bowls、piece→pieces、handful→handfuls、sandwich→sandwiches 等，没在表里的词不变。
- 份量不是数字开头（「干重 40 g」），或者是按热量描述的（「120 kcal 一份」），显示成「2 × 干重 40 g」。

**排序规则**：

- 主排序：近 7 天（含今天）里每样食物被记过的条数，从多到少。一次记 3 份也只算 1 次。
- 次数相同：英文界面按字母排，中文界面按拼音排，用 `Locale("zh-Hans")` 比较。
- 刷新时机：打开页面、切换餐次、切换分类时重排；连续加减时不重排，避免格子在手指底下跳动。

**新增 / 编辑食物、一次性记录**：

- 当前界面语言的名字必填，另一种语言选填。
- 另一种语言留空时，用 Translation 框架（`TranslationSession`）自动翻译名字和份量。品牌名、数字和单位保持原样；翻译失败照样保存，提示之后可以补。
- 热量可以填 0（零卡饮料），但不能空着。

**改目标页**：

- 训练日、休息日各一张卡片：热量（大号输入框），以及蛋白、碳水、脂肪。
- 今天适用的那张排在前面，并加标记。
- 每张卡片实时显示 4×蛋白 + 4×碳水 + 9×脂肪 合计多少热量（按当前单位），方便和热量目标对照。
- 页面里也能设置默认训练日是星期几。

**近 14 天**：

- 用 Swift Charts 画柱状图，每天一根柱子，加一条当天目标线，超标的柱子标红。
- 顶部显示近 7 天的日均热量和日均蛋白（只算有记录的天）。
- 点某一天，跳到那天的今日页。

**双语**：

- 界面文案放进 String Catalog（`Localizable.xcstrings`），语言有 English 和简体中文。
- App 内放一个语言切换按钮，选择存在 `@AppStorage` 里，通过 `.environment(\.locale, …)` 切换。
- 分类在数据里存英文键，显示时再翻译。

**热量单位（kcal / kJ）**：

- 数据库里一律存 kcal；kJ 只是显示和输入时的换算，1 kcal = 4.184 kJ。
- 选择存在 `@AppStorage("energyUnit")`，默认 kcal。
- 所有显示热量的地方都跟着换：大号数字、目标、剩余、顶部提示、食物格子、套餐、记录、近 14 天。营养成分表上另起一小行显示另一种单位的换算（「≈ 7,950 kJ」）。
- 新增 / 编辑食物和改目标时，输入框按当前单位显示和输入，保存时换算回 kcal（保留一位小数）。
- 防止来回换算产生误差：输入框记住打开时显示的值，如果用户没改过，就直接保存原来的 kcal，不用换算后的数。

**纯逻辑放进 `Logic/`，并写单元测试**：

- 合计
- 某天是训练日还是休息日
- 某天的目标
- 自动选餐次（覆盖 04:59 / 05:00 / 11:29 / 11:30 / 16:29 / 16:30 / 20:59 / 21:00 这几个边界）
- 近 7 天排行
- 份量随份数缩放（`scaleUnit`，用上面的例子做测试用例）
- kcal / kJ 换算和「未修改不换算」
- 日期加减

---

## 7. 里程碑（一次给 Claude Code 一个）

| # | 内容 | 完成标准 |
| --- | --- | --- |
| M0 | 环境与骨架：`project.yml`、App 入口、空的几个页面、Makefile、`ios/CLAUDE.md` | `make build` 通过，模拟器能打开 App |
| M1 | 数据层：第 4 节的模型、Targets 单例、种子导入、`Logic/` 和单元测试 | `make test` 全绿；首次启动导入 34 样食物和 2 个套餐 |
| M2 | 今日页：营养成分表、餐次、食物网格加减、记录列表、撤销 | 在模拟器里能完整记一天 |
| M3 | 管理：新增 / 编辑 / 删除食物、一次性记录、套餐、改目标页 | 网页版这些功能在 App 里都能用 |
| M4 | 近 14 天 + 图表 | 数字和网页版算法一致 |
| M5 | 双语 + 自动翻译 | 中英切换完整；留空的名字能自动翻译 |
| M6 | iCloud 同步（需付费账号） | 两台设备之间记录能同步 |
| M7 | 可选：HealthKit 写入、主屏小组件、快捷指令（比如「记一个鸡蛋」） | 按需 |
| M8 | 装到 iPhone：免费签名（7 天）或 TestFlight | 自己手机上每天能用 |

---

## 8. 以后想上架的话

- 加入 Apple Developer Program（99 美元/年）。
- 在 App Store Connect 建 App：填隐私标签（会涉及健康与健身数据）、准备截图、提交审核。
- 如果用了 HealthKit，审核会仔细看健康数据的用途说明；数据不能拿去做广告。
- 中国区 App Store 需要做 App 备案；只上美区等其他地区可以先不管。

---

## 9. 给 Claude Code 的第一条指令（示例）

> 读 `docs/IOS_PLAN.md` 和 `seed/` 里的 JSON。在 `ios/` 目录完成 M0：用 XcodeGen 建一个 SwiftUI 项目（iOS 18，bundle id `com.jiachengzhao.dietlog`，App 名 Diet Log / 饮食日志），写好 Makefile（gen / build / test / run），再写 `ios/CLAUDE.md` 记录项目约定（目录结构、SwiftData + CloudKit 约束、每步要跑 build 和 test）。完成后运行 `make build`，确认能在模拟器里启动。需要我在 Xcode 里手动操作的地方，告诉我点哪里。
