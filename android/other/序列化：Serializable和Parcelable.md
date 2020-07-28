# 前言
为了将数据进行跨平台存储、网络传输，推出`序列化`概念。我们知道这些操作方式是基于IO而言，而IO支持的格式是字节数组，故需要将对象转换为字节数组进行传输。
但如果只转换为没有规则的字节数组，那这些数据从IO中读取出来也是意义不大的，所以需要在对象转换为字节数组时制定一种规则，在IO读取数据时再用这种规则将对象还原。
所以序列化的意思就是`将对象转化成能进行跨平台或网络传输的字节序列`

Java中序列化操作只需实现`Serializable`接口即可，然后就可以将对象写入文件了；在Android中提供两种序列化方式：实现`Serializable`接口；实现`Parcelable`接口，重写代码。
这两种方式也常用来在activity间传递对象

# 实现Serializable接口
```java
public class Bean implements Serializable {
    
    private String a;
    private int b;
    
    public Bean() {
    }

    public Bean(String a, int b) {
        this.a = a;
        this.b = b;
    }
}
```
在Activity中传递
```java
Intent intent = new Intent(this, Main2Activity.class);
intent.putExtra("bean", new Bean());
startActivity(intent);
```
在另一个Activity中接收
```java
Bean bean = (Bean) getIntent().getSerializableExtra("bean");
```

# 实现Parcelable接口
```java
public class Bean implements Parcelable {
    ...
    //------下面是多出来的东西------
    
    protected Bean(Parcel in) {
        //这里的顺序要和writeToParcel方法内的顺序一致
        a = in.readString();
        b = in.readInt();
    }

    public static final Creator<Bean> CREATOR = new Creator<Bean>() {
        @Override
        public Bean createFromParcel(Parcel in) {
            return new Bean(in);
        }

        @Override
        public Bean[] newArray(int size) {
            return new Bean[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        //这里的顺序要和构造方法内的顺序一致
        dest.writeString(a);
        dest.writeInt(b);
    }
}
```
在Activity中传递不需要做修改

在另一个Activity中接收
```java
Bean bean = getIntent().getParcelableExtra("bean");
```

# 两种方式比较
- 缺点
  - `Serializable`方式使用反射原理，序列化时会产生大量临时变量，开销大，效率低
  - `Parcelable`方式只能对内存对象序列化，不能对需要存储到文件的对象序列化
- 优点
  - `Serializable`方式代码少，方便；能保证数据持久性和稳定性
  - `Parcelable`方式编译快，效率高，在传递集合时尤为明显

![image](https://img-blog.csdnimg.cn/20200728105322621.png)

大部分情况建议使用`Parcelable`，保存数据时建议使用`Serilalizable`
