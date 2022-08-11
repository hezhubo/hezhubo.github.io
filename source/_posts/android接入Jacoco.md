---
title: android接入Jacoco
date: 2022-06-14 09:37:59
tags: [android,测试]
categories: Android
---

> Jacoco是开源的Java代码覆盖率工具
>
> github地址：https://github.com/jacoco/jacoco

### 一、在项目工程*build.gradle*中添加插件依赖

```groovy
buildscript {
    ...
    
    dependencies {
        ...
        classpath "org.jacoco:org.jacoco.core:0.8.7"
    }
}
```

### 二、Module(application/library)中*build.gradle*添加插件配置

```groovy
apply plugin: 'jacoco'
jacoco {
    toolVersion = 0.8.7
}
android {
    ...
    
    buildTypes {
        debug {
            ...
            testCoverageEnabled = true // debug模式开启覆盖率统计
        }
    }
}
```

### 三、App Module中*build.gradle*添加task任务

```groovy
tasks.withType(Test) {
    jacoco.includeNoLocationClasses = true
    jacoco.excludes = ['jdk.internal.*'] // Allows it to run on Java 11
    // see related issue https://github.com/gradle/gradle/issues/5184#issuecomment-457865951
}

project.afterEvaluate {
    android.applicationVariants.all { variant ->
        // 由于我的应用含有多环境配置，因此会创建多个task
        if (variant.buildType.name == 'debug') {
            def variantName = variant.name
            def taskName = "jacoco${variantName}Report"

            tasks.create(name: "$taskName", type: JacocoReport) {
                group = "Reporting"
                description = "Generate Jacoco coverage reports."

                // 生成报表的类型
                reports {
                    html.enabled = true
                    xml.enabled = true
                    csv.enabled false
                }

                // 过滤掉不参与统计的代码
                def excludes = [
                        // data binding
                        'android/databinding/**/*.class',
                        '**/android/databinding/*Binding.class',
                        '**/android/databinding/*',
                        '**/androidx/databinding/*',
                        '**/BR.*',
                        // android
                        '**/R.class',
                        '**/R$*.class',
                        '**/BuildConfig.*',
                        '**/Manifest*.*',
                        'android/**/*.*',
                        // kotlin
                        '**/*MapperImpl*.*',
                        '**/*$ViewInjector*.*',
                        '**/*$ViewBinder*.*',
                        '**/BuildConfig.*',
                        '**/*Component*.*',
                        '**/*BR*.*',
                        '**/Manifest*.*',
                        '**/*$Lambda$*.*',
                        '**/*Hilt*.*',
                        '**/*MembersInjector*.*',
                        '**/*_MembersInjector.class',
                        '**/*_Factory*.*',
                        '**/*_Provide*Factory*.*',
                        '**/*Extensions*.*',
                ]

                // 编译字节码及源码路径配置
                def javaClasses = []
                def kotlinClasses = []
                def src = []

                javaClasses   << fileTree(dir: "$project.buildDir/intermediates/javac/${variantName}", excludes: excludes)
                kotlinClasses << fileTree(dir: "$project.buildDir/tmp/kotlin-classes/${variantName}", excludes: excludes)
                src           << "$project.projectDir/src/main/java"

                rootProject.subprojects.each { project ->
                    if (project.name != 'app') { // 处理子模块class路径（其中app为application module模块）
                        javaClasses   << fileTree(dir: "$project.buildDir/intermediates/javac/debug", excludes: excludes)
                        kotlinClasses << fileTree(dir: "$project.buildDir/tmp/kotlin-classes/debug", excludes: excludes)
                        src           << "$project.projectDir/src/main/java"
                    }
                }

                classDirectories.setFrom(files([
                        javaClasses,
                        kotlinClasses,
                ]))
                sourceDirectories.setFrom(files([
                        src
                ]))

                // 收集要分析的执行数据文件路径（把手机存储的数据文件复制到此目录下）
                def androidTestsData = fileTree(dir: "$rootDir/jacoco/", includes: ["**/*.ec"])
                executionData(files([
                        androidTestsData
                ]))
            }

        }

    }
}
```

### 四、写入报告到手机中(如下示例)

```kotlin
object JaCocoUtils {

    private var execFilePath = ""
    private var isWriting = false

    private fun writeExecFile() {
        if (isWriting) {
            return
        }
        Thread().run {
            if (execFilePath.isEmpty() || !File(execFilePath).exists()) {
                return@run
            }
            isWriting = true
            var out: OutputStream? = null
            try {
                out = FileOutputStream(execFilePath, false)
                // 通过反射获取报告内容
                val agent = Class.forName("org.jacoco.agent.rt.RT")
                    .getMethod("getAgent")
                    .invoke(null)
                out.write(
                    agent.javaClass.getMethod(
                        "getExecutionData",
                        Boolean::class.javaPrimitiveType
                    ).invoke(agent, false) as ByteArray
                )
            } catch (e: Exception) {
                Log.e("JaCocoUtils", e.toString(), e)
            } finally {
                if (out != null) {
                    try {
                        out.close()
                    } catch (e: IOException) {
                    }
                }
            }
            isWriting = false
        }
    }

    private fun createExecFile() {
        try {
            val file = File(execFilePath)
            if (file.exists()) {
                file.delete()
            }
            file.createNewFile()
            Log.i("JaCocoUtils", "jacoco exec file path = $execFilePath")
        } catch (e: Exception) {
            Log.e("JaCocoUtils", "jacoco exec file can not create! path = $execFilePath")
        }
    }

    @JvmStatic
    fun setJaCocoExecFilePath(path: String) {
        execFilePath = path
        createExecFile()
    }

}
```

### 五、生成报告

* 把手机中存储的报告文件拷贝至项目的**/jacoco**文件夹下（task任务中配置的目录，可自行更改调整）
* 执行对应的task。在gradle任务窗口，主工程app的task下，生成一个reporting目录，执行里面对应环境的task（task名称为**jacoco*XXXX*Report**），成功后在主工程**build/reports/jacoco/jacoco*XXXX*Report/**目录下查看报告

### 六、报告说明

| 字段                     | 名称       | 说明                                                         |
| ------------------------ | ---------- | ------------------------------------------------------------ |
| Element                  | 元素       | 展示分组名称：包名>类名>方法                                 |
| Missed Instructions Cov. | 指令覆盖   | 表明所有指令中，哪些被执行过哪些没被执行                     |
| Missed Branches Cov.     | 分支覆盖率 | 统计所有分支数量，同时指出哪些分支被执行，哪些没有被执行（分支点可映射到源码中每一行且高亮显示：红钻，无覆盖；黄钻，部分覆盖；绿钻，全覆盖） |
| Missed Cxty              | 圈复杂度   | 每个非抽象方法计算圈复杂度，并且也会计算每个类、包、组的复杂度。根据McCabe 1996的定义，圈复杂度可以理解为覆盖所有的可能情况最少使用的测试用例数 |
| Missed Methods           | 方法       | 一个方法至少执行了一条指令，则认为这个方法被执行过           |
| Missed Classes           | 类         | 每个类中只要有一个方法被执行，则认为这个类被执行             |
| Missed Lines             | 代码行     | 用背景色标记的都算行统计的目标，变量定义不算行，else也不算（红色背景，无覆盖；黄色背景，部分覆盖；绿色背景，全覆盖） |

