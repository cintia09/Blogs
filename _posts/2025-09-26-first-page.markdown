---
layout: post
title:  "Swift Data 迁移实战：我是如何优雅地为我的 App 数据升级的"
date:   2025-09-26 18:05:55 +0800
image:  04.jpg
tags:   第一篇。。。
---
这是我尝试写的第一篇博客。

# Swift Data 迁移实战：我是如何优雅地为我的 App 数据升级的

大家好！最近半年，我一直在写一个类似 ChatGPT 的 AI 聊天 App，平台是 macOS，技术栈是 SwiftUI + Swift Data。这个 App 的特点是能直接从 Hugging Face 下载模型，在 Mac 本地运行，也能接入 Ollama 和 Gemini 的 API，算是一个集成了不同后端的本地 AI 聊天工具。

在写这个 App 的过程中，我碰到了很多问题，觉得有必要写下来做一个记录和分享。如果你也碰到过相同的问题，那么这个分享就值了。

今天分享的第一个问题，就是 **Swift Data 的数据迁移**。

### 困境：数据积累与模型迭代的矛盾

我的 App 运行了一段时间后，会积累大量的数据，比如聊天记录、模型配置等等。但在开发过程中，我需要不断修改和优化 Model。问题来了：每次我修改完 Model，重新编译的 App 就会因为数据结构对不上而无法启动，除非把旧数据全部删掉。这意味着之前积累的所有聊天数据和配置都得付之一炬。

这在开发阶段还能忍，但对发布后的 App 来说是绝对无法接受的。我们总不能让用户每次更新 App，所有宝贵的数据就都没了吧？幸运的是， Swift Data 提供了数据迁移（Migration）功能。下面就是我是如何使用这个功能的一些心得。

（如果你还不是很了解 Swift Data，建议先看下 Apple 官方的视频，个人觉得那些视频讲解的还不错。）

### 核心问题：新旧代码与 Model 的“紧耦合”

数据迁移的基本思路，是保留上一个版本的 Model 定义，这样 Swift Data 才能知道如何从旧结构里读取数据，并把它转成新结构。

但这马上就带来了另一个问题：如果我代码里保留了旧的 `ConversationV0`，又定义了新的 `ConversationV1`，那么我的 SwiftUI 视图和其他业务逻辑代码到底该用哪个？如果统一用新的 `ConversationV1`，那么在迁移代码中引用旧 Model 的地方就会编译失败。这种代码和具体 Model 版本的“紧耦合”，让整个项目变得非常脆弱。



为了解决这个问题，我们需要让 App 的业务逻辑和具体的 Model 版本定义**解耦**。

### 我的解决方案：分层解耦架构

经过一番折腾，我设计了下面这套架构。核心思想有两个：

1.  **Schema 版本隔离**：把每个版本的 Model 定义，完全隔离在各自的 Schema 命名空间里。
2.  **外部接口统一**：通过 `typealias` 和 `extension`，给外部代码提供一个稳定、统一的 Model 接口。

听起来有点抽象？别急，我们一步步来看。

#### 第一步：用 `VersionedSchema` 把 Model “关起来”

首先，我们给每个数据模型版本创建一个遵循 `VersionedSchema` 协议的 `enum`。比如，我有 `SchemaV0` 和 `SchemaV1`。

然后，最关键的一步来了：**把每个版本的 Model 定义，都写在对应 Schema 枚举的 `extension` 里**。

```swift
// V0 版本定义
enum SchemaV0: VersionedSchema {
    static var versionIdentifier: Schema.Version { .init(0, 0, 0) }
    static var models: [any PersistentModel.Type] {
        [Conversation.self, Message.self, LLModelAPI.self]
    }
}

extension SchemaV0 {
    @Model
    final class Conversation {
        // ... V0 版本的属性定义
    }
    // ... 其他 V0 Model
}

// V1 版本定义
enum SchemaV1: VersionedSchema {
    static var versionIdentifier: Schema.Version { .init(1, 0, 0) }
    static var models: [any PersistentModel.Type] {
        [Conversation.self, Message.self, Model.self, ModelAPI.self]
    }
}

extension SchemaV1 {
    @Model
    final class Conversation {
        // ... V1 版本的属性定义（可能已经修改）
    }
    // ... 其他 V1 Model
}
```

这么做的好处是，`SchemaV0.Conversation` 和 `SchemaV1.Conversation` 现在是两个完全独立的类型，不会打架了。旧版本的 Model 被整整齐齐地“封装”在自己的版本号下面，井水不犯河水。



#### 第二步：用 `typealias` 搭建一座桥梁

现在，App 的其他部分（比如 SwiftUI 视图、ViewModel）怎么知道该用哪个版本的 Model 呢？答案是 `typealias`（类型别名）。

我创建了一个专门的文件 `AppData.swift`，在里面定义了当前 App 正在使用的 Model 类型。

```swift
import SwiftData

// 整个 App 的 Model 版本切换，只需要修改这里！
typealias Message = SchemaV1.Message
typealias Conversation = SchemaV1.Conversation
typealias ModelAPI = SchemaV1.ModelAPI
typealias Model = SchemaV1.Model
```

在 App 的所有其他代码中，我们不再关心具体的 `SchemaV1.Conversation`，而是直接使用 `Conversation` 这个别名。将来如果要升级到 `SchemaV2`，我只需要把 `typealias Conversation = SchemaV1.Conversation` 改成 `typealias Conversation = SchemaV2.Conversation`，整个 App 就无缝切换到了新版本，维护成本瞬间降低。



#### 第三步：用 `extension` 把“原始数据”翻译成“业务语言”

为了进一步解耦，我立了一个规矩：**在 Schema 内部定义的 Model，所有属性都必须是 Swift 的基本类型**（`String`, `Int`, `Data` 等），绝不使用任何自定义的 `enum` 或 `struct`。

那如果我想在代码里用 `ModelProvider` 这种自定义枚举怎么办？答案还是 `extension`。我们对 `typealias` 之后的类型进行扩展，添加计算属性和便利构造器。

```swift
// 我们是对外暴露的 'Conversation' 这个类型别名进行扩展
extension Conversation {
    // 计算属性，把底层的 String "翻译" 成业务逻辑中的枚举
    var modelProvider: ModelProvider {
        get {
            ModelProvider(rawValue: providerRaw) ?? .none
        }
        set {
            providerRaw = newValue.rawValue
        }
    }

    // 便利构造器，让外部代码可以用业务类型来创建实例
    convenience init(modelProvider: ModelProvider, ...) {
        self.init(providerRaw: modelProvider.rawValue, ...)
    }
}
```

通过这种方式，我们把 **数据存储层（只认原始类型）** 和 **业务逻辑层（使用自定义类型）** 完美地隔离开。Model 的核心定义只管怎么存数据，所有复杂的业务逻辑都封装在 `extension` 里。

### 开始动手：实现迁移计划

架构搭好了，实现迁移计划就水到渠成了。我们需要创建一个遵循 `SchemaMigrationPlan` 的类型，我叫它 `AppMigrationPlan`。

```swift
enum AppMigrationPlan: SchemaMigrationPlan {
    // 1. 告诉 Swift Data 我们有哪些版本的 Schema
    static var schemas: [any VersionedSchema.Type] {
        [
            SchemaV0.self,
            SchemaV1.self
        ]
    }

    // 2. 定义从一个版本到另一个版本的迁移阶段
    static var stages: [MigrationStage] {
        [
            migrateV0toV1
        ]
    }
}
```

迁移的核心是定义 `MigrationStage`。在我的 V0 到 V1 迁移中，我对 Model 做了比较大的重构，所以用了自定义迁移（`MigrationStage.custom`）。

```swift
extension AppMigrationPlan {
    static let migrateV0toV1 = MigrationStage.custom(
        fromVersion: SchemaV0.self,
        toVersion: SchemaV1.self,
        willMigrate: { context in
            // 1. 从旧版本数据库里，把所有数据捞出来
            let oldAPIs = try context.fetch(FetchDescriptor<SchemaV0.LLModelAPI>())

            // 2. 遍历旧数据，创建新版本的实例，然后把属性一个一个对应上
            for oldAPI in oldAPIs {
                let newAPI = SchemaV1.ModelAPI(
                    id: oldAPI.id,
                    providerRaw: oldAPI.providerRaw,
                    endPoint: oldAPI.endPoint,
                    bookmark: //...一些逻辑判断
                )
                // 3. 把新实例插入到上下文中
                context.insert(newAPI)
                // 4. 别忘了把旧实例删掉
                context.delete(oldAPI)
            }

            // ...对 Conversation 和 Message 也执行类似的操作...

            // 5. 全部搞定，保存！
            try context.save()
        },
        didMigrate: nil
    )
}
```

这个过程其实就像搬家：
1.  `willMigrate` 开始时，我们进入“旧房子”（`SchemaV0`）。
2.  用 `context.fetch` 把所有“家具”（数据）都清点出来。
3.  一件一件地搬到“新房子”（`SchemaV1`）里，摆到正确的位置上（属性映射）。
4.  搬完后，把“旧房子”拆了（`context.delete`）。
5.  最后 `context.save()` 确认搬家完成。

当 App 启动时，Swift Data 会自动检查数据版本和代码版本，发现对不上号，就会来找我们的 `AppMigrationPlan`，然后执行对应的 `stage`。整个过程对用户来说是完全无感的。

### 总结一下：这套方案的优缺点

搞完这一套，我认为这套基于版本化 Schema 和 `typealias` 解耦的架构，在处理 Swift Data 迁移时非常好用。

**优点：**

1.  **结构清晰，扩展性强**：每个 Schema 版本和它对应的 Model 都被整齐地放在一起。以后加 `SchemaV2`、`SchemaV3`，只要重复这个模式就行，项目不会乱。
2.  **高度解耦，容易维护**：业务代码和具体的数据模型版本彻底分开。升级 Model 时，大部分代码根本不用动，大大降低了维护成本和改出 bug 的风险。
3.  **编译时安全**：因为业务代码只认 `typealias` 定义的最新 Model，任何不兼容的修改在编译时就会报错，避免了上线后才发现问题的尴尬。
4.  **迁移逻辑集中管理**：所有的迁移步骤都集中在 `SchemaMigrationPlan` 里，一目了然。

**不足：**

1.  **有些模板代码**：每个版本都要维护一套完整的 Model 定义，即使只是改动一个很小的字段，也得把整个文件复制一份。这会增加一些代码量。
2.  **手动迁移有点繁琐**：对于复杂的自定义迁移，需要手写大量的属性映射代码。这个过程比较枯燥，而且容易出错（比如漏了一个字段没映射）。
3.  **前期有门槛**：相比直接写一个 `@Model` 类，这套架构需要更多前期的规划和设置。对一些特别简单的项目来说可能有点“用力过猛”。

总而言之，对于一个需要长期迭代、数据模型会不断演进的 App 来说，这套方案带来的清晰结构和高可维护性，完全值得前期的投入。它帮我解决了数据升级这个大难题，让我的 App 可以更稳健地向前发展。

希望这次的分享对你有用！如果你有更好的方法或者任何问题，欢迎留言交流。
