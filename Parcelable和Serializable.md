### 序列化的目的

------

  （1）永久的保存对象数据(将对象数据保存在文件当中,或者是磁盘中

  （2）通过序列化操作将对象数据在网络上进行传输(由于网络传输是以字节流的方式对数据进行传输的.因此序列化的目的是将对象数据转换成字节流的形式)

  （3）将对象数据在进程之间进行传递(Activity之间传递对象数据时,需要在当前的Activity中对对象数据进行序列化操作.在另一个Activity中需要进行反序列化操作讲数据取出)

  （4）Java平台允许我们在内存中创建可复用的Java对象，但一般情况下，只有当JVM处于运行时，这些对象才可能存在，即，这些对象的生命周期不会比JVM的生命周期更长（即每个对象都在JVM中）但在现实应用中，就可能要停止JVM运行，但有要保存某些指定的对象，并在将来重新读取被保存的对象。这是Java对象序列化就能够实现该功能。（可选择入数据库、或文件的形式保存）

  （5）序列化对象的时候只是针对变量进行序列化,不针对方法进行序列化.

  （6）在Intent之间,基本的数据类型直接进行相关传递即可,但是一旦数据类型比较复杂的时候,就需要进行序列化操作了.

 （7）序列化的对象和反序列化得到的对象可能内容一样，但本质上是两个对象。



### Serializable 接口

------

是Java提供的一个序列化接口，是一个空接口，为对象提供标准的序列化和反序列化操作。

#### serialVersionUID 

> **声明方法** ：

1.默认：private static final long  serialVersionUID =1L

2.系统根据当前类的结构自动生成它的hash值。如果当前类有所改变，比如增删了某些成员变量，那么系统会重新计算当前类的hash值并把它赋值给serialVersionUID。



> ps:

serialVersionUID不是必需的，不声明也可以实现序列化。但是对反序列化会产生影响。

**序列化是将对象的状态信息转换为可存储或传输的形式的过程。我们都知道，Java对象是保存在JVM的堆内存中的，也就是说，如果JVM堆不存在了，那么对象也就跟着消失了。**

而序列化提供了一种方案，可以让你在即使JVM停机的情况下也能把对象保存下来的方案。就像我们平时用的U盘一样。把Java对象序列化成可存储或传输的形式（如二进制流），比如保存在文件中。这样，当再次需要这个对象的时候，从文件中读取出二进制流，再从二进制流中反序列化出对象。

**虚拟机是否允许反序列化，不仅取决于类路径和功能代码是否一致，一个非常重要的一点是两个类的序列化 ID 是否一致**，这个所谓的序列化ID，就是我们在代码中定义的serialVersionUID



> serialVersionUID的工作机制

原则上序列化后的serialVersionUID和本地类（即序列化的那个类）的serialVersionUID相同才能正常地被序列化。

序列化的时候系统会把当前类的serialVersionUID写入序列化的文件中（也有可能是其他中介），当反序列化的时候，JVM会把传来的字节流中的serialVersionUID于本地相应实体类的serialVersionUID进行比较，看它是否和当前类的serialVersionUID一致。

一致则说明序列化类的版本和当前类的版本是相同的，这个时候序列化则可以成功。否则失败。

*注：* 如果类的结构发生非常规性的改变，比如修改了类名，修改了成员的类型，尽管serialVersionUID验证通过了，但是反序列化的过程还是会失败。因为类结构有了毁灭性的改变，根本无法从老版本的数据中还原出一个新的类结构的对象。

> serialVersionUID的blog : https://segmentfault.com/a/1190000038315377

**序列化User类**

```java
import java.io.Serializable;

public class User implements Serializable {
    private static final long serialVersionUID =1L;
    public int id;
    public String name;


    public User(int id, String name){
        this.id = id;
        this.name = name;
            }
}

```



**实现序列化的类SerialTestA**

```java
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.ObjectOutputStream;

public class SerialTestA {
    public static void main(String[] args) {
    User user=new User(1,"IU");
    ObjectOutputStream out;

    {
        try {
            out = new ObjectOutputStream(new FileOutputStream("cache.txt"));
            out.writeObject(user);
            out.close();

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

}}

```



**实现反序列化的类SerialTestB**

```java
import java.io.FileInputStream;
import java.io.IOException;
import java.io.ObjectInputStream;

public class SerialTestB {
    public static void main(String[] args) {
        try {
            ObjectInputStream in=new ObjectInputStream(new FileInputStream("cache.txt"));
            User newUser =(User) in.readObject();
            in.close();
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }

    }
}

```

**注意：**

> 1.当我们先执行SerialTestA，然后把User类的serialVersionUID改变，最后再执行SerialTestB，会出现java.io.InvalidClassException异常

> 2.当我们在User类中新增变量，再通过SerialTestA序列化，再把User类中新增的变量删除，最后通过SerialTestB反序列化。最后新增的变量会被SerialTestB忽视。

> 3.在SerialTestA正常序列化后，在User里新增变量，最后执行SerialTestB的反序列化。SerialTestB新增的变量被赋予默认值。



### Parcelable 接口

------

序列化的一种方式。实现了这个接口，一个类的对象就可以实现序列化并可以通过Intent或者Binder传递。



**用法**

```java
import android.os.Parcel;
import android.os.Parcelable;

public class Book implements Parcelable {

    private String name;
    private int id;
    private String classify;

    protected Book(Parcel in) {
        name = in.readString();
        classify = in.readString();
        id = in.readInt();
    }

    public Book(String classify, String name, int id) {
        this.name = name;
        this.id = id;
        this.classify = classify;
    }

    /**
     * 反序列化
     */
    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel in) {
            return new Book(in);//Creator与protected Book(Parcel in)配合实现反序列化，转换为对象。
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };

    //返回当前对象的内容描述。如果含有文件描述符，返回1。否则返回0，几乎所有情况都返回0。
    @Override
    public int describeContents() {
        return 0;
    }

    /**
     * 序列化过程
     *
     * @param dest
     * @param flags
     * flags有两种值：0或1。为1时标识当前对象要作为返回值返回，不能立即释放资源，几乎所有情况都为0。
     */
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
        dest.writeString(classify);
        dest.writeInt(id);
    }


    @Override
    public String toString() {
        return "name : " +
                name + "\"" + "id : " + id + "\"" + "classify" + classify;
    }
}
```

**Book类里面有其它对象：**

如果Book类里面有其他对象（比如实体类Data）的话，那么Data也需要实现Parcelable接口，用法与上面的Book类一样。

**writeToParcel**里面需要写上：**dest.writeParcelable(data, 0)**;

**protected Book(Parcel in) {}**里面需要写上**data = in.readParcelable(Data.class.getClassLoader());**



**MainActivity**中：

```java
textView.setOnClickListener(new View.OnClickListener() {
      @Override
      public void onClick(View v) {
          Intent intent = new Intent(MainActivity.this, Test1Activity.class);
          intent.putExtra("key", new Book("I", "U", 6));
          startActivity(intent);
      }
  });
```

**另一个Activity获取：**

```
Intent intent = getIntent();
Book book = intent.getParcelableExtra("key");
Log.d("Test1Activity", book.toString());
```



**系统已经为我们提供了许多实现了Parcelable接口的类，它们都是可以直接序列化的。比如Intent、Bundle、Bitmap，同时List和Map也可以序列化，前提是它们里面的每个元素都是可序列化的。**



### Serializable和Parcelable的区别

------

Serializable是Java中的序列化接口，其使用简单但是开销很大。序列化和反序列化过程需要大量I/O操作。而Parcelable是Android中的序列化方式，因此更适合用在Android平台上，缺点是使用起来稍微麻烦，但是效率很高，这是Android推荐的序列化方式。

Parcelable主要用在内存序列化上。

Serializable主要用于将对象序列化到存储设备中或者将对象序列化后通过网络传输。