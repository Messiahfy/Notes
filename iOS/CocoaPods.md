https://draven.co/cocoapods/
https://chuquan.me/2021/12/24/podfile-analyze-principle/
理解 Xcode 中的各种概念：https://chuquan.me/2021/12/03/understand-concepts-in-xcode/
理解 Xcode 中的各种文件：https://chuquan.me/2021/12/14/understand-files-in-xcode/

## 初始化
执行以下命令，会在当前目录（必须是Xcode项目）下生成Podfile文件
```
pod init
```

Podfile 文件使用 Ruby 脚本语言的语法

```
// 指定支持的平台和版本
platform :ios, '9.0'

// 定义要将它们链接到的 Xcode 目标
target 'myapp' do
  # Comment the next line if you don't want to use dynamic frameworks
  use_frameworks!

  // 可以在这里添加依赖库，比如 pod 'ObjectiveSugar'

  target 'myappTests' do
    inherit! :search_paths
    # Pods for testing
  end

  target 'myappUITests' do
    # Pods for testing
  end

end
```

## 执行
`pod install`

## 创建自己的pod
`pod lib create MyLibrary`

## Podfile API
https://guides.cocoapods.org/syntax/podfile.html#podfile

## 插件
1. 安装模板工具
gem install cocoapods-plugins

2. 生成插件（会创建目录结构）
pod plugins create NAME

## hook
plugin、pre_install、pre_integrate、post_install、post_integrate