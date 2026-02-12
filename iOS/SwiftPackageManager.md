官方文档：
* https://docs.swift.org/swiftpm/documentation
* https://developer.apple.com/documentation/xcode/swift-packages

# Swift Package
`Swift Package`是 Swift、Objective-C、Objective-C++、C 或 C++ 代码的可重用组件，可以包含源码、二进制文件和资源。每个 Swift 包都需要在包的主目录中有一个`Package.swift`文件，称为包清单。创建 Swift 包时，您可以使用包清单中的`PackageDescription`库来列出依赖项、配置本地化资源以及设置其他配置选项。包清单还允许您定义可执行产品，以及 Swift 包管理器可用于在清单中构建其他产品的插件。

示例：
```swift
import PackageDescription

let package = Package(
    name: "SlothCreator",
    platforms: [
        .macOS(.v11),
        .iOS(.v14),
        .watchOS(.v7),
        .tvOS(.v13)
    ],
    products: [
        .library(
            name: "SlothCreator",
            targets: ["SlothCreator"]
        )
    ],
    targets: [
        .target(
            name: "SlothCreator",
            resources: [
                .process("Resources/")
            ]
        )
    ]
)
```
`Package`的初始化函数用于初始化一个Swift Package：
```swift
init(
    name: String, // 包的名称，如果使用git url推断名称，则传nil
    defaultLocalization: LanguageTag? = nil, // 默认的语言，用于本地化资源
    platforms: [SupportedPlatform]? = nil, // 支持的平台
    pkgConfig: String? = nil, // pkg-config 文件名，用于系统库集成（如 Linux）。
    providers: [SystemPackageProvider]? = nil, // 系统包提供者，如 .brew("openssl") 或 .apt("libfoo-dev")），帮助在不同 OS 上安装系统依赖。
    products: [Product] = [], // 对外提供的产品，比如 .library、.executable、.plugin，产品会关联 targets
    dependencies: [Package.Dependency] = [], // 包的依赖。可以是远程（URL + 版本要求）或本地（path）。
    targets: [Target] = [], // 包内部的构建单元，代表一组源代码、资源或其他文件的集合
    swiftLanguageModes: [SwiftLanguageMode]? = nil, // Swift 语言版本，如 .v5、.v6。
    cLanguageStandard: CLanguageStandard? = nil, // C 语言标准（如 .c11）
    cxxLanguageStandard: CXXLanguageStandard? = nil // C++ 语言标准（如 .cxx14）
)
```
Products 是由一个或多个 targets 构建的。换句话说，products 是 targets 的“打包形式”，对外暴露。每个 product 必须关联至少一个 target，target 提供实际的代码和资源。一个 target 可以被多个 products 使用（例如，一个核心 target 可以同时用于静态库和动态库产品）。

```swift
let package = Package(
    name: "MyPackage",
    products: [
        .library(name: "MyLibrary", targets: ["CoreModule", "Utils"]), // 产品依赖两个目标
        .executable(name: "MyApp", targets: ["AppMain"])
    ],
    targets: [
        .target(name: "CoreModule", dependencies: ["Utils"]), // 目标间依赖
        .target(name: "Utils", dependencies: []),
        .executableTarget(name: "AppMain", dependencies: ["CoreModule"]),
        .testTarget(name: "CoreModuleTests", dependencies: ["CoreModule"])
    ]
)
```

`Package`的`dependencies`是包级别的依赖，定义在 Package 初始化函数的参数中，描述了当前包依赖的外部包（如 Git 仓库中的其他 Swift 包）或本地包（本地文件路径）。它指定了整个包需要引入的外部资源。


Target 的 dependencies 是目标级别的依赖，可以依赖包内的其他 target，也可以依赖Package.dependencies 中定义的外部包。
```swift
import PackageDescription

let package = Package(
    name: "MyMath",
    products: [
        .library(name: "MathLibrary", targets: ["MathLib"]),
        .executable(name: "MathCLI", targets: ["MathCLI"])
    ],
    dependencies: [
        // 包级别：声明外部包
        .package(url: "https://github.com/apple/swift-numerics.git", from: "1.0.0")
    ],
    targets: [
        // 目标级别：MathLib 依赖 swift-numerics 的 RealModule 产品
        .target(
            name: "MathLib",
            dependencies: [
                .product(name: "RealModule", package: "swift-numerics")
            ]
        ),
        // MathCLI 依赖 MathLib（内部目标）和 RealModule（外部产品）
        .executableTarget(
            name: "MathCLI",
            dependencies: [
                "MathLib",
                .product(name: "RealModule", package: "swift-numerics")
            ]
        ),
        .testTarget(
            name: "MathLibTests",
            dependencies: ["MathLib"] // 测试只依赖 MathLib
        )
    ]
)
```

创建包：
```
mkdir MyPackage
cd MyPackage
swift package init --type library  # 可以执行包则使用 executable
```
或者 打开 Xcode 并选择“文件”>“新建>包”
