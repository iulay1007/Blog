### SharedPreferences

------

使用SharedPreferences(**保存用户偏好参数**)保存数据， 当我们的应用想要保存用户的一些偏好参数，***比如是否自动登陆，是否记住账号密码,是否在Wifi下才能 联网等相关信息***,如果使用数据库的话,显得有点大材小用了！我们把上面这些配置信息称为用户的偏好 设置，就是用户偏好的设置，而这些配置信息通常是保存在特定的文件中！比如windows使用ini文件， 而J2SE中使用properties属性文件与xml文件来保存软件的配置信息;而在Android中我们通常使用 一个轻量级的存储类——SharedPreferences来保存用户偏好的参数！



*SharedPreferences是Android平台上一个轻量级的存储类，用来保存应用的一些常用配置，比如Activity状态，Activity暂停时，**将此activity的状态保存到SharedPereferences中；当Activity重载，系统回调方法onSaveInstanceState时，再从SharedPreferences中将值取出***



* SharedPreferences是一种轻量级的数据存储方式，采用键值对的存储方式。

* SharedPreferences**只能存储少量数据**，大量数据不能使用该方式存储，支持存储的数据类型有booleans, floats, ints, longs, and strings。

* SharedPreferences**存储到一个XML文件中的**，路径在**/data/data/<packagename>/shared_prefs/**下



  **在xml文件中保存形式如下**

  ```xml
  <?xml version='1.0' encoding='utf-8' standalone='yes' ?>
  <map>
      <float name="isFloat" value="1.5" />
      <string name="isString">Android</string>
      <int name="isInt" value="1" />
      <long name="isLong" value="1000" />
      <boolean name="isBoolean" value="true" />
      <set name="isStringSet">
          <string>element 1</string>
          <string>element 2</string>
          <string>element 3</string>
      </set>
  </map>
  ```



  #### 性能

  * ShredPreferences是单例对象，第一次打开后，之后获取都无需创建，速度很快。

  - 当第一次获取数据后，数据会被加载到一个缓存的Map中，之后的读取都会非常快。
  - 当由于是XML<->Map的存储方式，所以，数据越大，操作越慢，get、commit、apply、remove、clear都会受影响，所以尽量把数据按功能拆分成若干份。



  #### 获取SharedPreferences对象

  要创建存储文件或访问已有数据，首先要获取SharedPreferences才能进行操作。获取SharedPreferences对象有下面两个方式：

  （1）**Context.getSharedPreferences**(String name, int mode) --- **通过Context调用该方法获得对象**。它有两个参数，第一个name 指定了SharedPreferences存储的文件的文件名，第二个参数mode 指定了操作的模式。这种方式获取的对象创建的文件 可以被整个应用所有组件使用，有指定的文件名。

  > SharedPreferences类似过去Windows系统上的ini配置文件，但是它分为多种权限，可以全局共享访问。
  >
  > mode：模式，包括
  > MODE_PRIVATE：默认模式。只能被自己的应用程序访问
  >
  > MODE_MULTI_PRIVATE：用于多个进程共同操作一个SharedPreferences文件
  >
  > MODE_WORLD_READABLE：除了自己访问外还可以被其它应该程序读取
  >
  > MODE_WORLD_WRITEABLE：除了自己访问外还可以被其它应该程序读取和写入
  >
  > 后两种基本废弃

  （2）**Activity.getPreferences**(int mode) --- **通过Activity调用获得对象**。它只有一个参数mode 指定操作模式。这种方式获取的对象创建的文件 属于Activity,只能在该Activity中使用，且没有指定的文件名，文件名同Activity名字。

    (3)**referenceManager.getDefaultSharedPreferences(Context)**
  使用这个方法会自动使用当前程序的包名作为前缀来命名SharedPreferences文件



  #### 示例代码

  ```java
      /**
       * 保存用户信息
       */
      private void saveUserInfo(){
          SharedPreferences userInfo = getSharedPreferences(PREFS_NAME, MODE_PRIVATE);
          SharedPreferences.Editor editor = userInfo.edit();//获取Editor
          //得到Editor后，写入需要保存的数据
          editor.putString("username", "一只猫的涵养");
          editor.putInt("age", 20);
          editor.commit();//提交修改
          Log.i(TAG, "保存用户信息成功");
      }
      /**
       * 读取用户信息
       */
      private void getUserInfo(){
          SharedPreferences userInfo = getSharedPreferences(PREFS_NAME, MODE_PRIVATE);
          String username = userInfo.getString("username", null);//读取username
          int age = userInfo.getInt("age", 0);//读取age
          Log.i(TAG, "读取用户信息");
          Log.i(TAG, "username:" + username + "， age:" + age);
      }
      /**
       * 移除年龄信数据
       */
      private void removeUserInfo(){
          SharedPreferences userInfo = getSharedPreferences(PREFS_NAME, MODE_PRIVATE);
          SharedPreferences.Editor editor = userInfo.edit();//获取Editor
          editor.remove("age");
          editor.commit();
          Log.i(TAG, "移除年龄数据");
      }
  
      /**
       * 清空数据
       */
      private void clearUserInfo(){
          SharedPreferences userInfo = getSharedPreferences(PREFS_NAME, MODE_PRIVATE);
          SharedPreferences.Editor editor = userInfo.edit();//获取Editor
          editor.clear();
          editor.commit();
          Log.i(TAG, "清空数据");
      }
  ```

  **在这些动作之后，记得commit**

```java
editor.putInt("age", 20);//写入操作
editor.remove("age");   //移除操作
editor.clear();     //清空操作
editor.commit();//记得commit
```



#### 局限性

系统对它的读/写有一定的缓存策略，即在内存中会有一份SharedPreferences文件的缓存，因此在多进程模式下系统对它的读/写就变得不可靠，当面对高并发的读/写访问SharedPreferences有很大几率会丢失数据，因此，不建议在进程通信中使用SharedPreferences。