#### 方式一：buildSrc

新建Java module创建名为buildSrc，名为buildSrc的插件会自动加入到编译路径，不需要做额外的配置。

1、只保留src/main目录及gradle文件，其余文件删除；
 2、在src/main路径下新建groovy目录（java编写的话，就新建java目录，groovy编写就在groovy目录下，否则会失败）；
 3、在src/main路径下新建如下目录：/resources/META-INF/gradle-plugins

修改gradle文件如下：

```java
apply plugin: 'groovy'

dependencies {
    implementation localGroovy()
    implementation gradleApi()
    implementation fileTree(dir: 'libs', include: ['*.jar'])
}

sourceCompatibility = "7"
targetCompatibility = "7"

```

下面我们在groovy目录下新建包，自定义一个简单的插件：

```java
package com.wzh.gradle

import org.gradle.api.Action
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.api.Task

public class LocalDemoPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {
        project.task("local",new Action<Task>() {
            @Override
            void execute(Task task) {
                println("task "+task.getName()+" is running! ")
            }
        })

    }
}
```

然后在/resources/META-INF/gradle-plugins目录下新建一个配置文件localgradle.properties，内容如下：

 ```java
#这里面的class就是我们插件
implementation-class=com.wzh.gradle.LocalDemoPlugin
 ```

接下来在app模块使用插件：

```java
apply plugin: 'localgradle'
```

build以下，我们会看到输出的日志：

> task local is running!

我们可以在插件里面进行一些操作，比如可以用TransformAPI对字节码文件进行修改等。

