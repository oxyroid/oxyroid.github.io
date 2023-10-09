---
title: 使用 lint 执行代码检查
date: 2023-10-09 12:15:51
tags: [ kotlin, lint, tool ]
---

### 前言

前段时间我完善了我的一个开源项目的有关贡献项目的说明文档，要求贡献者严格遵循规范提交代码。 但是绝大多数规范只能寄托于开发者自行遵守，无法自动检查。于是我就有了使用 `lint` 强制规范代码的想法。

### lint

又叫 `lint check` 或者 `lint tool`， 他可以检测潜在错误，以及在正确性、安全性、性能、易用性、便利性和国际化方面是否需要优化改进， 帮助我们发现代码质量问题，同时提供一些解决方案。每个问题都有信息描述和等级。

同样的检查工具还有 `CheckStyle`, `Ktlint`, `Detekt` 等。`CheckStyle` 只支持 `Java` 因此我们不讨论，后两者为现在主流的 `kotlin` 代码检查器，我们以后会讨论。

### 需求

本文我将实现：不允许 star-import，以及不允许调用 WindowInsetsCompat 的某个函数。

### 准备

1. 新建一个java module，用于存放 lint 规则
2. 配置 build.gradle.kts

```kotlin
plugins {
    `java-library`
    id("kotlin")
    id("com.android.lint")
}
dependencies {
    compileOnly("org.jetbrains.kotlin:kotlin-stdlib-jdk8")
    compileOnly("com.android.tools.lint:lint-api:<lint_version>")
}
```

3. 新建一个 IssueRegistry，用于注册 lint

```kotlin
// 一个 lint module 只存在一个 IssueRegistry
class CustomIssueRegistry : IssueRegistry() {
    // 这里可以指定多个不同的检查
    override val issues: List<Issue> = TODO()
}
```

4. 与 IssueRegistry 注册表关联

```kotlin
// 新建注册表
// resources/META-INF/services/com.android.tools.lint.client.api.IssueRegistry
// 只能指定一个 IssueRegistry
com.`<package>`.CustomIssueRegistry
```

### 开始实现

#### NonStarImportsDetector: 不允许 star-import

先了解以下概念：

- Issue: 问题，直观上讲就是呈现给用户（编码者）的警告/报错/日志，可以指定优先级或者附带快速修复等。
- IssueRegistry: 问题注册器，顾名思义，用于注册本 lint check 中所有的问题，同时也能声明出处信息（作者，网站，联系方式等）。每个
  lint module 只允许存在一个 IssueRegistry，并且在注册表中注册。
- Detector: 探测器，可以扫描各类问题，是 Issue 定位问题的实现载体。
- Scanner: 扫描器，对 Detector 检查范围进行约束，它提供了,例如只用于检查 源代码，AndroidManifest 等。

```kotlin
class StaredImportsDetector : Detector(), SourceCodeScanner {
    override fun getApplicableUastTypes() = listOf(UImportStatement::class.java)

    override fun createUastHandler(context: JavaContext) = object : UElementHandler() {
        override fun visitImportStatement(node: UImportStatement) {
            val reference = node.importReference ?: return
            if (node.isOnDemand) {
                context.report(
                    issue = StaredImportsIssue,
                    scope = reference,
                    location = context.getLocation(reference),
                    message = "Using special import instead of star-import."
                )
            }
        }
    }

    companion object {
        val StaredImportsIssue = Issue.create(
            id = "StaredImportsIssue",
            briefDescription = "using special imports instead",
            explanation = "using special imports instead of stared imports",
            category = Category.CORRECTNESS,
            severity = Severity.WARNING,
            implementation = Implementation(
                StaredImportsDetector::class.java,
                EnumSet.of(Scope.JAVA_FILE, Scope.TEST_SOURCES)
            )
        )
    }
}
```