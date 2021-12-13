
### 一、认识Gradle
#### 构建工具

---

**构建工具**用于实现项目自动化，是一种可编程的工具，你可以用代码来控制构建流程最终生成可交付的软件。构建工具可以帮助你创建一个重复的、可靠的、无需手动介入的、不依赖于特定操作系统和IDE的构建。

先看下Android apk构建流程官方提供的流程图：

![image](/Users/liujian/Pictures/20180130145051521.png)

这个APK构建的过程主要分为以下几步：

1. 通过AAPT(Android Asset Packaging Tool)打包res资源文件，比如AndroidManifest.xml、xml布局文件等，并将这些xml文件编译为二进制，其中assets和raw文件夹的文件不会被编译为二进制，最终会生成R.java和resources.arsc文件。
2. AIDL工具会将所有的aidl接口转化为对应的Java接口。
3. 所有的Java代码，包括R.java和Java接口都会被Java编译器编译成.class文件。
4. Dex工具会将上一步生成的.class文件、第三库和其他.class文件编译成.dex文件。
5. 上一步编译生成的.dex文件、编译过的资源、无需编译的资源（如图片等）会被ApkBuilder工具打包成APK文件。
6. 使用Debug Keystore或者Release Keystore对上一步生成的APK文件进行签名。
7. 如果是对APK正式签名，还需要使用zipalign工具对APK进行对齐操作，这样应用运行时会减少内存的开销。

从以上步骤可以看出，APK的构建过程是比较繁琐的，而且这个构建过程又是时常重复的，手动去完成构建工作，无疑对于开发人员是个折磨，也会产生诸多的问题，导致项目开发周期变长。这个时候就需要**构建工具**的帮助。

Gradle便是一款强大的**构建工具**，组成可细分为三个方面：
1. `groovy `：包括 groovy 基本语法、闭包、数据结构、面向对象等等。
2. `Android DSL（build scrpit block）`：Android 插件在 Gradle 所特有的东西，我们可以在不同的 build scrpit block 中去做不同的事情。
3. `Gradle API`：包含 Project、Task、Setting 等等（本文重点）。

#### Gradle的优势 
---

##### 1、更好的灵活性
在灵活性上，Gradle 相对于 Maven、Ant 等构建工具， 其 提供了一系列的 API 让我们有能力去修改或定制项目的构建过程。例如我们可以 利用 Gradle 去动态修改生成的 APK 包名，但是如果是使用的 Maven、Ant 等工具，我们就必须等生成 APK 后，再手动去修改 APK 的名称。

##### 2、更细的粒度
在粒度性上，使用 Maven、Ant 等构建工具时，我们的源代码和构建脚本是独立的，而且我们也不知道其内部的处理是怎样的。但是，我们的 Gradle 则不同，它 从源代码的编译、资源的编译、再到生成 APK 的过程中都是一个接一个来执行的。    
此外，Gradle 构建的粒度细化到了每一个 task 之中。并且它所有的 Task 源码都是开源的，在我们掌握了这一整套打包流程后，我们就可以通过去修改它的 Task 去动态改变其执行流程。例如 Tinker 框架的实现过程中，它通过动态地修改 Gradle 的打包过程生成 APK 的同时，也生成了各种补丁文件。

##### 3、更好的扩展性
在扩展性上，Gradle 支持插件机制，所以我们可以复用这些插件，就如同复用库一样简单方便。
##### 4、更强的兼容性
Gradle可以和Ant、Maven和Ivy进行集成，比如我们可以把Ant的构建脚本导入到Gradle的构建中。

### 二、Gradle构建生命周期
Gradle 的构建过程分为 三部分：初始化阶段、配置阶段和执行阶段。

![image](/Users/liujian/Pictures/ac8dt-yct0f.png)

- **1）**、首先，解析 settings.gradle 来获取模块信息，这是初始化阶段。
- **2）**、然后，配置每个模块，配置的时候并不会执行 task。
- **3）**、接着，配置完了以后，有一个重要的回调 project.afterEvaluate，它表示所有的模块都已经配置完了，可以准备执行 task 了。
- **4）**、最后，执行指定的 task 及其依赖的 task。


#### 1、初始化阶段

---

首先，在这个阶段中，会读取根工程中的 setting.gradle 中的 include 信息，确定有多少工程加入构建，然后，会为每一个项目（build.gradle 脚本文件）创建一个个与之对应的 Project 实例，最终形成一个项目的层次结构。
与初始化阶段相关的脚本文件是 settings.gradle，而一个 settings.gradle 脚本对应一个 Settings 对象，我们最常用来声明项目的层次结构的 include 就是 Settings 对象下的一个方法，在 Gradle 初始化的时候会构造一个 Settings 实例对象，以执行各个 Project 的初始化配置。   

##### settings.gradle
在 settings.gradle 文件中，我们可以 在 Gradle 的构建过程中添加各个生命周期节点监听，其代码如下所示：

```
include ':app'
rootProject.name='GradleTestDemo'

gradle.addBuildListener(new BuildListener() {
    @Override
    void buildStarted(Gradle gradle) {
        println("开始构建")
    }

    @Override
    void settingsEvaluated(Settings settings) {
        println("setting评估完成（setting.gradle 中代码执行完毕）")
    }

    @Override
    void projectsLoaded(Gradle gradle) {
        println("项目加载完成（初始化阶段结束）")
        println("初始化阶段结束，可访问根项目" + gradle.rootProject)
    }

    @Override
    void projectsEvaluated(Gradle gradle) {
        println("所以项目评估完成（配置阶段结束）")
    }

    @Override
    void buildFinished(BuildResult result) {
        println("构建完成")
    }
})
```
执行下Make Project，Build Output可看到输出结果：

```
Executing tasks: [:app:assembleDebug] in project /Users/liujian/Documents/study/androiddemo/GradleTestDemo

setting评估完成（setting.gradle 中代码执行完毕）
项目加载完成（初始化阶段结束）
初始化阶段结束，可访问根项目root project 'GradleTestDemo'
所以项目评估完成（配置阶段结束）
> Task :app:preBuild UP-TO-DATE
...
> Task :app:assembleDebug UP-TO-DATE
构建完成

BUILD SUCCESSFUL in 0s
25 actionable tasks: 25 up-to-date
```

在 settings.gradle 文件中，可以指定其它 project 的位置，将其它外部工程中的 moudle 导入到当前的工程之中：

```
// 导入其它 App 的 speech 语音模块
include "speech"
project(":speech").projectDir = new File("../OtherApp/speech")
```

#### 2、配置阶段

配置阶段的任务是 执行各项目下的 build.gradle 脚本，完成 Project 的配置，与此同时，会构造 Task 任务依赖关系图以便在执行阶段按照依赖关系执行  Task。而在配置阶段执行的代码通常来说都会包括以下三个部分的内容，如下所示：

- 1)、build.gralde 中的各种语句。
- 2)、**闭包**。
- 3)、 Task中的配置段语句。

需要注意的是，执行任何 Gradle 命令，在初始化阶段和配置阶段的代码都会被执行。

#### 3、执行阶段

在配置阶段结束后，Gradle 会根据各个任务 Task 的依赖关系来创建一个有向无环图，我们可以通过 Gradle 对象的 getTaskGraph 方法来得到该有向无环图 => TaskExecutionGraph，并且，当有向无环图构建完成之后，所有 Task 执行之前，我们可以通过 whenReady(groovy.lang.Closure) 或者addTaskExecutionGraphListener(TaskExecutionGraphListener) 来接收相应的通知：

```
gradle.getTaskGraph().addTaskExecutionGraphListener(new TaskExecutionGraphListener() {
    @Override
    void graphPopulated(TaskExecutionGraph graph) {
    }
})
```

然后，Gradle 构建系统会通过调用 gradle <任务名> 来执行相应的各个任务。

#### 构建各个阶段、任务的耗时情况

---

了解了 Gradle 生命周期中的各个 Hook 方法之后，我们就可以 **利用它们来获取项目构建各个阶段、任务的耗时情况**，在 settings.gradle 中加入如下代码即可:

```
long beginOfSetting = System.currentTimeMillis()
def beginOfConfig
def configHasBegin = false
def beginOfProjectConfig = new HashMap()
def beginOfProjectExcute = System.currentTimeMillis()
gradle.projectsLoaded {
    println '初始化阶段，耗时：' + (System.currentTimeMillis() -
            beginOfSetting) + 'ms'
}
gradle.beforeProject { project ->
    if (!configHasBegin) {
        configHasBegin = true
        beginOfConfig = System.currentTimeMillis()
    }
    beginOfProjectConfig.put(project, System.currentTimeMillis())
}
gradle.afterProject { project ->
    def begin = beginOfProjectConfig.get(project)
    println '配置阶段，' + project + '耗时：' +
            (System.currentTimeMillis() - begin) + 'ms'
}
gradle.taskGraph.whenReady {
    println '配置阶段，总共耗时：' + (System.currentTimeMillis() -
            beginOfConfig) + 'ms'
    beginOfProjectExcute = System.currentTimeMillis()
}
gradle.taskGraph.beforeTask { task ->
    task.doFirst {
        task.ext.beginOfTask = System.currentTimeMillis()
    }
    task.doLast {
        println '执行阶段，' + task + '耗时：' +
                (System.currentTimeMillis() - task.beginOfTask) + 'ms'
    }
}
gradle.buildFinished {
    println '执行阶段，耗时：' + (System.currentTimeMillis() -
            beginOfProjectExcute) + 'ms'
}
```

在 Gradle 中，执行每一种类型的配置脚本就会创建与之对应的实例，而在 Gradle 中如 **三种类型的配置脚本**，如下所示：

- 1）、`Build Scrpit`：**对应一个 Project 实例，即每个 build.gradle 都会转换成一个 Project 实例**。
- 2）、`Init Scrpit`：**对应一个 Gradle 实例，它在构建初始化时创建，整个构建执行过程中以单例形式存在**。
- 3）、`Settings Scrpit`：**对应一个 Settings 实例，即每个 settings.gradle 都会转换成一个 Settings 实例**。

可以看到，一个 Gradle 构建流程中会由一至多个 project 实例构成，而每一个 project 实例又是由一至多个 task 构成。下面，来认识下 Project、task。

### 三、project、task、sourceSet

在 Project 中有很多的 API，但是根据它们的 **属性和用途** 我们可以将其分解为 **六大部分**，如下图所示：

![Image](/Users/liujian/Pictures/111.png)

对于 Project 中各个部分的作用，可以先来大致了解下：

1. `Project API`：让当前的 Project 拥有了操作它的父 Project 以及管理它的子 Project 的能力。
2. `Task 相关 API`：为当前 Project 提供了新增 Task 以及管理已有 Task 的能力。
3. `Project 属性相关的 Api`：Gradle 会预先为我们提供一些 Project 属性，而属性相关的 api 让我们拥有了为 Project 添加额外属性的能力。
4. `File 相关 Api`：Project File 相关的 API 主要用来操作我们当前 Project 下的一些文件处理。
5. `Gradle 生命周期 API`：即上一章讲的生命周期 API。
6. `其它 API`：添加依赖、添加配置、引入外部文件等等零散 API 的聚合。

#### 1、Project相关

##### 1）、Project Api

          ###### getAllprojects

getAllprojects 表示 **获取所有 project 的实例**，示例代码如下所示：

```
/**
 * getAllProjects 使用示例
 */
this.getProjects()

def getProjects() {
    println "<================>"
    println " Root Project Start "
    println "<================>"
    // 1、getAllprojects 方法返回一个包含根 project 与其子 project 的 Set 集合
    // eachWithIndex 方法用于遍历集合、数组等可迭代的容器，
    // 并同时返回下标，不同于 each 方法仅返回 project
    this.getAllprojects().eachWithIndex { Project project, int index ->
        // 2、下标为 0，表明当前遍历的是 rootProject
        if (index == 0) {
            println "Root Project is $project"
        } else {
            println "child Project is $project"
        }
    }
}
```

rootProject 与其旗下的各个子工程组成了一个树形结构，但是这颗树的高度也仅仅被限定为了两层。

---

###### getSubprojects

getSubprojects 表示获取当前工程下所有子工程的实例，示例代码如下所示：

```
/**
 * getAllsubproject 使用示例
 */
this.getSubProjects()

def getSubProjects() {
    println "<================>"
    println " Sub Project Start "
    println "<================>"
    // getSubprojects 方法返回一个包含子 project 的 Set 集合
    this.getSubprojects().each { Project project ->
        println "child Project is $project"
    }
}

```

---

###### getParent

getParent 表示 获取当前 project 的父类，需要注意的是，如果我们在根工程中使用它，获取的父类会为 null，因为根工程没有父类。

---

###### getRootProject

使用 getRootProject 可在任意 build.gradle 文件获取当前根工程的 project 实例。

---

###### project

project 表示的是 指定工程的实例，然后在闭包中对其进行操作，如下配置指定工程插件引入：

```
/**
 * project 使用示例
 */

// 1、闭包参数可以放在括号外面
project("app") { Project project ->
    apply plugin: 'com.android.application'
}

// 2、更简洁的写法是这样的：省略参数
project("app") {
    apply plugin: 'com.android.application'
}
```

---

###### allprojects

allprojects 表示 **用于配置当前 project 及其旗下的每一个子 project**，如下所示：

```
/**
 * allprojects 使用示例
 */

// 同 project 一样的更简洁写法
allprojects {
    repositories {
        google()
        jcenter()
        mavenCentral()
        maven {
            url "https://jitpack.io"
        }
        maven { url "https://plugins.gradle.org/m2/" }
    }
}
```

在 allprojects 中我们一般用来配置一些通用的配置，比如上面最常见的全局仓库配置

---

###### subprojects

subprojects 可以 统一配置当前 project 下的所有子 project，示例代码如下所示:

```
/**
 * subprojects 使用示例：
 *    给所有的子工程引入 将 aar 文件上传置 Maven 服务器的配置脚本
 */
subprojects {
    if (project.plugins.hasPlugin("com.android.library")) {
        apply from: '../publishToMaven.gradle'
    }
}
```

在上述示例代码中，会先判断当前 project 旗下的子 project 是不是库，如果是库才有必要引入 publishToMaven 脚本。

---

##### 2）、Project 属性相关的 Api

###### ext扩展属性

Gradle 提供了 ext 关键字让我们有能力去定义自身所需要的扩展属性，可以通过它对工程中的依赖进行全局配置。目前比较流行的配置方式：

```
/**
 * 新建version.gradle配置文件
 */
def versions = [:]
versions.kotlin = "1.3.61"

def support = [:]
support.app_compat = 'androidx.appcompat:appcompat:1.0.0'
support.cardview = 'androidx.cardview:cardview:1.0.0'
support.customtabs = 'androidx.browser:browser:1.0.0'

def kotlin = [:]
kotlin.stdlib = "org.jetbrains.kotlin:kotlin-stdlib-jdk7:$versions.kotlin"
kotlin.reflect = "org.jetbrains.kotlin:kotlin-reflect:$versions.kotlin"

def deps = [:]
deps.support = support
deps.kotlin = kotlin

//使用ext字段扩展属性
ext.deps = deps
```

在dependencies依赖：

```
dependencies {
    implementation deps.support.app_compat
    implementation deps.support.cardview
    implementation deps.support.customtabs
    
    implementation deps.kotlin.stdlib
    implementation deps.kotlin.reflect
}
```

或者使用遍历的方式进行依赖：

```
dependencies {
    deps.support.each {k, v -> implementation v}
    deps.kotlin.each {k, v -> implementation v}
}
```

---

###### gradle.properties 定义扩展属性

除了使用 ext 扩展属性定义额外的属性之外，也可以在 gradle.properties 下定义扩展属性，其示例代码如下所示：

```
/**
 * gradle.properties中配置
 */
COMPILE_SDK_VERSION=29
MIN_SDK_VERSION=16

// 在 app moudle 下的 build.gradle 中使用
compileSdkVersion COMPILE_SDK_VERSION as int
minSdkVersion MIN_SDK_VERSION as int
```

##### 3）、File 相关 Api

###### 路径获取 

路径获取API主要有getRootDir()，getProjectDir()，getBuildDir()，其示例代码如下所示：

```
/**
 * 路径获取 API
 */
println "the root file path is:" + getRootDir().absolutePath
println "this build file path is:" + getBuildDir().absolutePath
println "this Project file path is:" + getProjectDir().absolutePath



/**
 * 输出结果
 */
 > Configure project :
the root file path is:/Users/liujian/Documents/study/androiddemo/GradleTestDemo
this build file path is:/Users/liujian/Documents/study/androiddemo/GradleTestDemo/build
this Project file path is:/Users/liujian/Documents/study/androiddemo/GradleTestDemo
```

---

###### 文件操作相关

`文件定位`: 常用的文件定位 API 有 `file/files`，其示例代码如下所示：

```
// 在 rootProject 下的 build.gradle 中

/**
 * 1、文件定位之 file
 */
this.getContent("config.gradle")

def getContent(String path) {
    try {
        // 不同与 new file 的需要传入 绝对路径 的方式，
        // file 从相对于当前的 project 工程开始查找
        def mFile = file(path)
        println mFile.text 
    } catch (GradleException e) {
        println e.toString()
        return null
    }
}

/**
 * 1、文件定位之 files
 */
this.getContent("config.gradle", "build.gradle")

def getContent(String path1, String path2) {
    try {
        // 不同与 new file 的需要传入 绝对路径 的方式，
        // file 从相对于当前的 project 工程开始查找
        def mFiles = files(path1, path2)
        println mFiles[0].text + mFiles[1].text
    } catch (GradleException e) {
        println e.toString()
        return null
    }
}

```

`文件拷贝`: 常用的文件拷贝 API 为 `copy`，其示例代码如下所示：

```
/**
 * 文件拷贝
 */
copy {
    // 既可以拷贝文件，也可以拷贝文件夹
    // 这里是将 app moudle 下生成的 apk 目录拷贝到
    // 根工程下的 build 目录
    from file("build/outputs/apk")
    into getRootProject().getBuildDir().path + "/apk/"
    exclude {
        // 排除不需要拷贝的文件
    }
    rename {
        // 对拷贝过来的文件进行重命名
    }
}
```

`文件树遍历`: 可以 使用 fileTree 将当前目录转换为文件数的形式，然后便可以获取到每一个树元素（节点）进行相应的操作，其示例代码如下所示：

```
/**
 * 文件树遍历
 */
fileTree("build/outputs/apk") { FileTree fileTree ->
    fileTree.visit { FileTreeElement fileTreeElement ->
        println "The file is $fileTreeElement.file.name"
        copy {
            from fileTreeElement.file
            into getRootProject().getBuildDir().path + "/apkTree/"
        }
    }
}
```

##### 4）、其它 API

###### 依赖相关 

分为根项目下的 buildscript 和 app moudle 下的 dependencies

```
/**
 * 根项目下的 buildscript
 */
buildscript {
    ext.kotlin_version = '1.3.50'
    repositories {
        google()
        jcenter()
        
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.3'
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

/**
 * app moudle 下的 dependencies
 */
implementation(rootProject.ext.dependencies.glide) {
        // 排除依赖：一般用于解决资源、代码冲突相关的问题
        exclude module: 'support-v4' 
        // 传递依赖：A => B => C ，B 中使用到了 C 中的依赖，
        // 且 A 依赖于 B，如果打开传递依赖，则 A 能使用到 B 中所使用的 C 中的依赖，
        //默认都是不打开，即 false
        transitive false 
}
```

###### 外部命令执行

一般是 使用 Gradle 提供的 exec 来执行外部命令，如下使用 exec 命令来 将当前工程下新生产的 APK 文件拷贝到 电脑下的 Downloads 目录中，示例代码如下所示：

```
/**
 * 使用 exec 执行外部命令
 */
task apkMove() {
    doLast { 
        // 在 gradle 的执行阶段去执行
        def sourcePath = this.buildDir.path + "/outputs/apk/speed/release/"
        def destinationPath = "/Users/liujian/Downloads/"
        def command = "mv -f $sourcePath $destinationPath"
        exec {
            try {
                executable "bash"
                args "-c", command
                println "The command execute is success"
            } catch (GradleException e) {
                println "The command execute is failed"
            }
        }
    }
}
```

#### 2、Task相关

gradle中所有的构建工作都是由task完成的,它帮我们处理了很多工作,比如编译,打包,发布等都是task。可以在项目的根目录下,打开命令行执行`gradlew tasks`查看当前项目所有的task。

先创建和执行一个简单的Task：

```
// 1、声明一个名为 TastTest 的 gradle task
task TastTest {
    // 2、在 TastTest task 闭包内输出 hello~，
    // 执行在 gradle 生命周期的第二个阶段，即配置阶段。
    println("hello~")
    // 3、给 task 附带一些 执行动作（Action），执行在gradle 生命周期的第三个阶段，即执行阶段。
    //doFirst：表示 task 执行最开始的时候被调用的 Action。
    //doLast：表示 task 将执行完的时候被调用的 Action。
    doFirst {
        println("start~~~")
    }
    doLast {
        println("end~~~")
    }
}
```

在terminal中执行 ./gradlew TastTest可看到输出结果：

```
Root Project is root project 'GradleTestDemo'
child Project is project ':app'
hello~
配置阶段，root project 'GradleTestDemo'耗时：581ms

> Configure project :app
配置阶段，project ':app'耗时：34ms
配置阶段，总共耗时：663ms

> Task :TastTest
start~~~
end~~~
执行阶段，task ':TastTest'耗时：0ms
执行阶段，耗时：4ms
```

---

###### Task定义

Task 常见的定义方式有 两种，示例代码如下所示：

```
// Task 定义方式1：直接通过 task 函数去创建（在 "()" 可以不指定 group 与 description 属性）
task myTask1(group: "MyTask", description: "task1") {
    println "This is myTask1"
}

// Task 定义方式2：通过 TaskContainer 去创建 task
this.tasks.create(name: "myTask2") {
    setGroup("MyTask")
    setDescription("task2")
    println "This is myTask2"
}
```

---

###### Task属性

不管是哪一种 task 的定义方式，在 "()" 内我们都可以配置它的一系列属性，目前 官方所支持的属性 可以总结为如下表格：

| 选型            | 描述                                                      | 默认值       |
| --------------- | :-------------------------------------------------------- | ------------ |
| name            | task 名字                                                 | 无，必须指定 |
| type            | 需要创建的 task Class                                     | DefaultTask  |
| action          | 当 task 执行的时候，需要执行的闭包 closure 或 行为 Action | null         |
| overwrite       | 替换一个已存在的 task                                     | false        |
| dependsOn       | 该 task 所依赖的 task 集合                                | []           |
| group           | 该 task 所属组                                            | null         |
| description     | task 的描述信息                                           | null         |
| constructorArgs | 传递到 task Class 构造器中的参数                          | null         |

此外可使用 ext 给 task 自定义需要的属性：

```
task Gradle_First() {
    ext.good = true
}

task Gradle_Last() {
    doFirst {
        //输出自定义属性
        println Gradle_First.good
    }
    doLast {
        //使用 "$" 来引用另一个 task 的属性
        println "I am not $Gradle_First.name"
    }
}
```

---

###### Task 的依赖和执行顺序

dependsOn可配置强依赖，设置task执行顺序。dependsOn 强依赖的方式可以细分为 **静态依赖和动态依赖**，示例代码如下所示：

```
task task1 {
    doLast {
        println "This is task1"
    }
}

task task2 {
    doLast {
        println "This is task2"
    }
}

// Task 静态依赖方式1 (常用）
task task3(dependsOn: [task1, task2]) {
    doLast {
        println "This is task3"
    }
}

// Task 静态依赖方式2
task3.dependsOn(task1, task2)

// Task 动态依赖方式
task dytask4 {
    dependsOn this.tasks.findAll { task ->
        return task.name.startsWith("task")
    }
    doLast {
        println "This is task4"
    }
}
```

执行./gradlew task3可看到task执行顺序为task1 -> task2 -> task3。

mustRunAfter可配合dependsOn使用，上述代码在task3中加入：

```
task task3(dependsOn: [task1, task2]) {
    mustRunAfter task2
    doLast {
        println "This is task3"
    }
}
```

即可改变执行顺序为task2 -> task1 -> task3。

---

###### Task 类型

除了定义一个新的 task 之外，我们也可以使用 type 属性来直接使用一个已有的 task 类型，比如 Gradle 自带的 `Copy、Delete、Sync task` 等等。示例代码如下所示：

```
// 1、删除根目录下的 build 文件
task clean(type: Delete) {
    delete rootProject.buildDir
}
// 2、将 doc 复制到 build/target 目录下
task copyDocs(type: Copy) {
    from 'src/main/doc'
    into 'build/target/doc'
}
// 3、执行时会复制源文件到目标目录，然后从目标目录删除所有非复制文件
task syncFile(type:Sync) {
    from 'src/main/doc'
    into 'build/target/doc'
}
```

#### 3、 SourceSet

SourceSet 主要是 **用来设置我们项目中源码或资源的位置的**，目前它最常见的两个使用案例就是如下 **两类**：

- 1）、**修改 so 库存放位置**。
- 2）、**资源文件分包存放**。

##### 1、修改 so 库存放位置

在 app moudle 下的 android 闭包下配置如下代码即可修改 so 库存放位置：

```
android {
    ...
    sourceSets {
        main {
            // 修改 so 库存放位置
            jniLibs.srcDirs = ["libs"]
        }
    }
}
```

##### 2、资源文件分包存放

同样，在 app moudle 下的 android 闭包下配置如下代码即可将资源文件进行分包存放：

```
android {
    sourceSets {
        main {
            res.srcDirs = ["src/main/res",
                           "src/main/res-play",
                           "src/main/res-shop"
                            ... 
                           ]
        }
    }
}
```

### 四、Gradle命令

Gradle 的命令有很多，但是我们通常只会使用如下两种类型的命令：

- 1）、获取构建信息的命令。
- 2）、执行 task 的命令。

##### 1、获取构建信息的命令

```
// 1、按自顶向下的结构列出子项目的名称列表
./gradlew projects
// 2、分类列出项目中所有的任务
./gradlew tasks
// 3、列出项目的依赖列表
./gradlew dependencies
```

##### 2、执行 task 的命令

常规的用于执行 task 的命令有 **四种**，如下所示：

```
// 1、用于执行多个 task 任务
./gradlew Test Gradle_Last
// 2、使用 -x 排除单个 task 任务
./gradlew -x Test
// 3、使用 -continue 可以在构建失败后继续执行下面的构建命令
./gradlew -continue Test
// 4、建议使用简化的 task name 去执行 task，下面的命令用于执行 
// Gradle_Last 这个 task
./gradlew G_Last
```

而对于子目录下定义的 task，我们通常会使用如下的命令来执行它：

```
// 1、使用 -b 执行 app 目录下定义的 task
./gradlew -b app/build.gradle MyTask
// 2、在大型项目中我们一般使用更加智能的 -p 来替代 -b
./gradlew -p app MyTask
```

### 五、Gradle插件

##### 自定义gradle插件

https://www.jianshu.com/p/b00ca8715bdc

##### gradle插件结合ASM实现代码插桩

https://www.jianshu.com/p/067b1399158d

