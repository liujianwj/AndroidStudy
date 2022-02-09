https://guolin.blog.csdn.net/article/details/109787732

### Dagger2（依赖注入框架）

目的：能够注入到任何你想要的对象，可以减少程序中对象间的耦合度

IOC控制反转：
是原来由程序中主动获取的资源，转变由第三方获取并使原来的代码被动接收的方式，以达到解耦的效果，称为控制反转。

例如：在每个类中都直接通过Student s = new Student()创建Student对象，如果有一天Student构造方法修改，则需要在对应每个类中进行修改，违背了开闭原则（对扩展开发，对修改关闭）

Dagger2在module中统一创建对象，在Component中将对象和需要注入的对象进行注入。
```java
//第一步 添加@Module 注解
@Module
public class MainModule {
    //第二步 使用Provider 注解 实例化对象
    @Provides
    Student providerStudent() {
        return new Student();
    }
}

//第一步 添加@Component
//第二步 添加module
@Component(modules = {MainModule.class})
public interface MainComponent {
    //第三步  写一个方法 绑定Activity /Fragment
    void inject(MainActivity activity);
}

//Rebuild Project，这里APT会帮我们在编译的时候自动生成代码，生成DaggerMainConponent类，DaggerMainConponent继承MainComponent，实现了inject，将MainActivity中添加有Inject注解的属性，通过MainModule对象获取。

public class MainActivity extends AppCompatActivity {
    /***
     * 第二步  使用Inject 注解，获取到A 对象的实例
     */
    @Inject
    Student s;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        /***
         * 第一步 添加依赖关系
         */
        //第一种方式
        DaggerMainConponent.create().inject(this);

        //第二种方式
        DaggerMainConponent.builder().build().inject(this);

        /***
         * 第三步  调用Student对象的方法
         */
        s.study();
    }
}
```
好处：
- 隔离，相对于对Student对象进行了封装包裹管理，发生变换的只有一个地方
- 职责分离，原先目标需要去管理Student的生产关系，现在统一交给Dagger2处理

### Hilt 

Hilt是对Dagger2的二次封装，不用写大量的Component，Hilt一共内置了7种组件类型，分别用于注入到不同的场景

<img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220120153029511.png" alt="image-20220120153029511" style="zoom:50%;" />