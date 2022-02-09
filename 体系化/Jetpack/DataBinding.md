#### 使用方法

```java
// 在build.gradle中开启DataBinding
dataBinding { enabled = true }

//model
public class User extends BaseObservable {
    private String name;
    private String pwd;

    public User(String name, String pwd) {
        this.name = name;
        this.pwd = pwd;
    }

    @Bindable
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
        notifyPropertyChanged(BR.name);
    }
}

//activity_main.xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"> <!--这里面就用来定义数据源--> 
      <data>

    <variable
        name="user"
        type="com.example.databinding.User" />
      </data>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <TextView
            android:id="@+id/tv1"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
               　<!-- =表示双向绑定的view->model，即修改tv1，model的值也会改变 -->
            android:text="@={user.name}"
            android:textSize="50sp"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintLeft_toLeftOf="parent"
            app:layout_constraintRight_toRightOf="parent"
            app:layout_constraintTop_toTopOf="parent" />
    </LinearLayout>
</layout>
              
              
//MainActivity.java
ActivityMainBinding binding;
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    binding= DataBindingUtil.setContentView(this, R.layout.activity_main);
  user=new User("Derry");
  binding.setUser(user); // 必须要建立绑定关系，否则没有任何效果
  user.setName(user.getName() + "口"); // view.setText(text);
}  
```



源码分析：

技术：apt技术

在编译期，通过apt技术，扫描到<layout></layout>包裹布局，会自动生成两个xml，一个是根据<data></data>生成，另一个是就是Android正常加载到xml布局。然后通过tag关联，即每个需要操作的控件都会被自动加上tag，第一个XML中都有tag对应。

Databinding根据第一个xml，生成各种与控件对应的对象，这里是操作卡的原因之一，然后通过设置runnable监听Molde的变化，去改变view，监听view的变化，去改变model。