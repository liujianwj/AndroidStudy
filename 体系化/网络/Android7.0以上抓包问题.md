### 遇到的问题是：

Android6.0及以下系统可以抓包，而Android7.0及以上系统不能再抓包。

### 原因：

Android7.0+的版本新增了证书验证，即app内不再像原来一样默认信任用户的证书。

### 怎么解决：

默认信任所有证书。

### 具体怎么实现:

1、在res底下创建一个xml文件夹，然后在内部创建一个名为 “network_security_config.xml”的文件，文件内容如下:



```xml
<network-security-config>
    <base-config cleartextTrafficPermitted="true">
        <trust-anchors>
            <certificates src="system" overridePins="true" />
            <certificates src="user" overridePins="true" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

2、在AndroidManifest里的<application>标签中，添加代码：



```bash
android:networkSecurityConfig="@xml/network_security_config"
```

然后重新编译打包即可成功抓包，困扰测试姐姐们已久的问题解决起来却如此简单。

### 添加以上的配置可能带来的风险或问题：

**1、安全问题：**
 开启上述配置后虽然可以成功抓包，但不可避免的也会带来一些异常风险问题，换句话说就是，我们希望自己人可以抓到包，但又不希望外部的人员抓到包。
 所以转换成需求就是：**在测试、开发阶段可以抓包，但正式发布包不能抓包。**
 怎么办呢？
 1、将前面AndroidManifest里的<application>标签中新增的代码改成



```bash
android:networkSecurityConfig="${NETWORK_SECURITY_CONFIG}"
```

相当于由前面的静态引入证书文件改成了动态引入,然后
 2、buildType中的release加入



```bash
release {
       manifestPlaceholders = [
           NETWORK_SECURITY_CONFIG: ""
       ]
}      
```

buildType中的debug中加入



```bash
debug {
       manifestPlaceholders = [
           NETWORK_SECURITY_CONFIG: "@xml/network_security_config"
       ]
}        
```

因为压根就没有引入证书文件，所以Release模式下打的包不能抓包，反之Debug包则可以。所以我们要做的就是开发测试阶段打Debug包、发布线上就打Release包即可。
 其实，除了利用Release和Debug的包类型实现区分可不可抓包，使用环境域名中的base_url来决定也可以，判断方式有很多，具体用哪种好，怎么实现就需要自己去摸索了。

### 2、android7.0以上的手机，开着网络代理访问不了webview页面

我们需要在webview的WebViewClient中，将下面这行代码给注释掉



```css
super.onReceivedSslError(view, handler, error);
```

这一段代码是为了忽略掉SSL证书错误，因为开启代理后网络会变得不安全，证书会错误，webview检测到证书错误之后就直接让webview白板，不请求任何数据。 这一节是为了忽略掉父类的处理，然后默认走下去。（网上的一些文章说的，实际没有遇到）

这个问题能说的就这么多了～




链接：https://www.jianshu.com/p/392362115090
