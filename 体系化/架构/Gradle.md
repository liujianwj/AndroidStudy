1、gradle wrapper

```java
目录
-工程
  -gradle 
    -wrapper
       -gradle-wrapper.jar
       -gradle-wrapper.properties
  
gradle-wrapper.properties:
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-5.1.1-all.zip
```

用于配置当前使用的gradle版本，如果本地有对应的版本，则不用下载，否则会去下载

2、根目录下build.gradle

```groovy
//构建脚本
buildscript {
  //构建所需要的插件所在仓库
  repositories {
        google()
        jcenter()
   }
  //构建所需要的插件，在module中的build.gradle引入，如apply plugin: 'com.android.application'
  dependencies{
    classpath 'com.android.tools.build:gradle:3.4.0'
  }
}

//module中dependencies依赖所在的仓库
allprojects{
  repositories {
        google()
        jcenter()
    }
}
```

3、闭包

