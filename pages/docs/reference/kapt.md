---
type: doc
layout: reference
title: "使用 kapt"
---

# Kotlin 注解处理

> 译注：kapt 即 Kotlin annotation processing tool（Kotlin 注解处理工具）缩写。

在 Kotlin 中通过 *kapt* 编译器插件支持注解处理器（参见[JSR 269](https://jcp.org/en/jsr/detail?id=269)）。

简而言之，你可以在 Kotlin 项目中使用像 [Dagger](https://google.github.io/dagger/) 或者 [Data Binding](https://developer.android.com/topic/libraries/data-binding/index.html) 这样的库。

关于如何将 *kapt* 插件应用于 Gradle/Maven 构建中，请阅读下文。

## 在 Gradle 中使用

应用 `kotlin-kapt` Gradle 插件：

<div class="multi-language-sample" data-lang="groovy">
<div class="sample" markdown="1" mode="groovy" theme="idea" data-lang="groovy">

```groovy
plugins {
    id "org.jetbrains.kotlin.kapt" version "{{ site.data.releases.latest.version }}"
}
```

</div>
</div>

<div class="multi-language-sample" data-lang="kotlin">
<div class="sample" markdown="1" mode="kotlin" theme="idea" data-lang="kotlin" data-highlight-only>

```kotlin
plugins {
    kotlin("kapt") version "{{ site.data.releases.latest.version }}"
}
```

</div>
</div>

或者使用 `apply plugin` 语法：

<div class="sample" markdown="1" mode="groovy" theme="idea">

```groovy
apply plugin: 'kotlin-kapt'
```

</div>

然后在 `dependencies` 块中使用 `kapt` 配置添加相应的依赖项：

<div class="multi-language-sample" data-lang="groovy">
<div class="sample" markdown="1" mode="groovy" theme="idea" data-lang="groovy">

```groovy
dependencies {
    kapt 'groupId:artifactId:版本'
}
```

</div>
</div>

<div class="multi-language-sample" data-lang="kotlin">
<div class="sample" markdown="1" mode="kotlin" theme="idea" data-lang="kotlin" data-highlight-only>

```kotlin
dependencies {
    kapt("groupId:artifactId:版本")
}
```

</div>
</div>

如果你以前使用 [Android 支持](https://developer.android.com/studio/build/gradle-plugin-3-0-0-migration.html#annotationProcessor_config)作为注解处理器，那么以 `kapt` 取代 `annotationProcessor` 配置的使用。如果你的项目包含 Java 类，`kapt` 也会顾全到它们。

如果为 `androidTest` 或 `test` 源代码使用注解处理器，那么相应的 `kapt` 配置名为 `kaptAndroidTest` 和 `kaptTest`。请注意 `kaptAndroidTest` 和 `kaptTest` 扩展了 `kapt`，所以你可以只提供 `kapt` 依赖而它对生产和测试源代码都可用。

## 注解处理器参数

使用 `arguments {}` 块将参数传给注解处理器：

<div class="sample" markdown="1" mode="groovy" theme="idea">

```groovy
kapt {
    arguments {
        arg("key", "value")
    }
}
```

</div>

{:#gradle-构建缓存支持自-1220-起}

## Gradle 构建缓存支持（自 1.2.20 起）

默认情况下，kapt 注解处理任务就会[在 Gradle 中缓存](https://guides.gradle.org/using-build-cache/)。注解处理器所运行的任意代码可能不一定将输入转换为输出、可能访问与修改 Gradle 未跟踪的文件等。If the annotation processors used in the build cannot be properly cached, it is possible to disable caching for kapt entirely by adding the following lines to the build script, in order to avoid false-positive cache hits for the kapt tasks:

<div class="sample" markdown="1" mode="groovy" theme="idea">

```groovy
kapt {
    useBuildCache = false
}
```

</div>

## 并行运行 kapt 任务（自 1.2.60 起）

To improve the speed of builds that use kapt, you can enable the [Gradle worker API](https://guides.gradle.org/using-the-worker-api/) for kapt tasks.
Using the worker API lets Gradle run independent annotation processing tasks from a single project in parallel, which in some cases significantly decreases the execution time.
However, running kapt with Gradle worker API enabled can result in increased memory consumption due to parallel execution.

To use the Gradle worker API for parallel execution of kapt tasks, add this line to your `gradle.properties` file:

<div class="sample" markdown="1" mode="xml" theme="idea">

```
kapt.use.worker.api=true
```

</div>

## kapt 的避免编译（自 1.3.20 起）

To improve the times of incremental builds with kapt, it can use the Gradle [compile avoidance](https://docs.gradle.org/current/userguide/java_plugin.html#sec:java_compile_avoidance).
With compile avoidance enabled, Gradle can skip annotation processing when rebuilding a project. Particularly, annotation processing is skipped when:
* The project's source files are unchanged.
* The changes in dependencies are [ABI](https://en.wikipedia.org/wiki/Application_binary_interface) compatible. For example, the only changes are in method bodies.

However, compile avoidance can't be used for annotation processors discovered in the compile classpath since _any changes_ in them require running the annotation processing tasks.

To run kapt with compile avoidance:
* Add the annotation processor dependencies to the `kapt*` configurations manually as described [above](#在-gradle-中使用).
* Turn off the discovery of annotation processors in the compile classpath by adding this line to your `gradle.properties` file:

<div class="sample" markdown="1" mode="xml" theme="idea">

```
kapt.include.compile.classpath=false
```

</div>

## 增量注解处理（自 1.3.30 起）

Starting from version 1.3.30, kapt supports incremental annotation processing as an experimental feature.
Currently, annotation processing can be incremental only if all annotation processors being used are incremental.

To enable incremental annotation processing, add this line to your `gradle.properties` file:

<div class="sample" markdown="1" mode="xml" theme="idea">

```
kapt.incremental.apt=true
```

</div>

Note that incremental annotation processing requires [incremental compilation](using-gradle.html#增量编译) to be enabled as well.

## Java 编译器选项

Kapt 使用 Java 编译器来运行注解处理器。
以下是将任意选项传给 javac 的方式：

<div class="sample" markdown="1" mode="groovy" theme="idea">

```groovy
kapt {
    javacOptions {
        // 增加注解处理器的最大错误次数
        // 默认为 100。
        option("-Xmaxerrs", 500)
    }
}
```

</div>

## 非存在类型校正

一些注解处理器（如 `AutoFactory`）依赖于声明签名中的精确类型。默认情况下，Kapt 将每个未知类型（包括生成的类的类型）替换为 `NonExistentClass`，但你可以更改此行为。将额外标志添加到 `build.gradle` 文件以启用在存根（stub）中推断出的错误类型：


<div class="sample" markdown="1" mode="groovy" theme="idea">

```groovy
kapt {
    correctErrorTypes = true
}
```

</div>

## 在 Maven 中使用

在 `compile` 之前在 kotlin-maven-plugin 中添加 `kapt` 目标的执行：

<div class="sample" markdown="1" mode="xml" auto-indent="false" theme="idea" data-highlight-only>

```xml
<execution>
    <id>kapt</id>
    <goals>
        <goal>kapt</goal>
    </goals>
    <configuration>
        <sourceDirs>
            <sourceDir>src/main/kotlin</sourceDir>
            <sourceDir>src/main/java</sourceDir>
        </sourceDirs>
        <annotationProcessorPaths>
            <!-- 在此处指定你的注解处理器。 -->
            <annotationProcessorPath>
                <groupId>com.google.dagger</groupId>
                <artifactId>dagger-compiler</artifactId>
                <version>2.9</version>
            </annotationProcessorPath>
        </annotationProcessorPaths>
    </configuration>
</execution>
```

</div>

你可以在
[Kotlin 示例版本库](https://github.com/JetBrains/kotlin-examples/tree/master/maven/dagger-maven-example) 中找到一个显示使用 Kotlin、Maven 和 Dagger 的完整示例项目。
 
请注意，IntelliJ IDEA 自身的构建系统目前还不支持 kapt。当你想要重新运行注解处理时，请从“Maven Projects”工具栏启动构建。


## 在命令行中使用

Kapt 编译器插件已随 Kotlin 编译器的二进制发行版分发。

可以使用 kotlinc 选项 `Xplugin` 提供该 JAR 文件的路径来附加该插件：

<div class="sample" markdown="1" mode="shell" theme="idea">

```bash
-Xplugin=$KOTLIN_HOME/lib/kotlin-annotation-processing.jar
```

</div>

以下是可用选项的列表：

* `sources`（*必需*）：所生成文件的输出路径。
* `classes`（*必需*）：所生成类文件与资源的输出路径。
* `stubs`（*必需*）：存根文件的输出路径。换句话说，一些临时目录。
* `incrementalData`：二进制存根的输出路径。
* `apclasspath`（*可重复*）：注解处理器 JAR 包路径。如果有的多个 JAR 包就传多个 `apclasspath` 选项。
* `apoptions`：注解处理器选项的 base64 编码列表。详见 [AP/javac options encoding](#apjavac-选项编码)。
* `javacArguments`：传给 javac 的选项的 base64 编码列表。详见 [AP/javac options encoding](#apjavac-选项编码)。
* `processors`：逗号分隔的注解处理器全类名列表。如果指定，kapt 就不会尝试在 `apclasspath` 中查找注解处理器。
* `verbose`：启用详细输出。
* `aptMode`（*必需*）
    * `stubs`——只生成注解处理所需的存根；
    * `apt`——只运行注解处理；
    * `stubsAndApt`——生成存根并运行注解处理。
* `correctErrorTypes`：参见[下文](#在-gradle-中使用)。默认未启用。

插件选项格式为：`-P plugin:<plugin id>:<key>=<value>`。选项可以重复。

一个示例：

<div class="sample" markdown="1" mode="shell" theme="idea">

```bash
-P plugin:org.jetbrains.kotlin.kapt3:sources=build/kapt/sources
-P plugin:org.jetbrains.kotlin.kapt3:classes=build/kapt/classes
-P plugin:org.jetbrains.kotlin.kapt3:stubs=build/kapt/stubs

-P plugin:org.jetbrains.kotlin.kapt3:apclasspath=lib/ap.jar
-P plugin:org.jetbrains.kotlin.kapt3:apclasspath=lib/anotherAp.jar

-P plugin:org.jetbrains.kotlin.kapt3:correctErrorTypes=true
```

</div>

## 生成 Kotlin 代码

Kapt 可生成 Kotlin 代码。是将生成的 Kotlin 源文件写入`processingEnv.options["kapt.kotlin.generated"]` 所指定的目录，这些文件会与主源代码一起编译。

可以在 [kotlin-examples](https://github.com/JetBrains/kotlin-examples/tree/master/gradle/kotlin-code-generation) Github 版本库中找到完整的示例。

请注意，对于所生成 Kotlin 文件，Kapt 不支持多轮处理。


## AP/javac 选项编码

`apoptions` 与 `javacArguments` 命令行选项接受选项编码映射。
这是自己编码选项的方式：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun encodeList(options: Map<String, String>): String {
    val os = ByteArrayOutputStream()
    val oos = ObjectOutputStream(os)

    oos.writeInt(options.size)
    for ((key, value) in options.entries) {
        oos.writeUTF(key)
        oos.writeUTF(value)
    }

    oos.flush()
    return Base64.getEncoder().encodeToString(os.toByteArray())
}
```

</div>
