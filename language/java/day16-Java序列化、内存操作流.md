[TOC]
# Java序列化、反序列化
## Java序列化
+ Java序列化是指把Java对象转换为字节序列的过程
+ Java反序列化是指把字节序列恢复为Java对象的过程

当两个Java进程进行通信时，发送方需要把这个Java对象转换为字节序列，然后在网络上传送；另一方面，接收方需要从字节序列中恢复出Java对象
    
**Java对象序列化注意事项：要被序列化的对象所在的类必须实现Serializable接口**
```java
import java.io.FileOutputStream;
import java.io.ObjectOutputStream;
public class ObjectOuputStreamDemo {
    public static void main(String[] args) {
        Person p = new Person("小明", 8, '男');
        // public ObjectOutputStream(OutputStream out)创建写入指定 OutputStream 的
        // ObjectOutputStream
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("D:/person.txt"));) {
            // public final void writeObject(Object obj)将指定的对象写入
            // ObjectOutputStream
            oos.writeObject(p);
            System.out.println("java对象序列化成功");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}    
```

```
import java.io.Serializable;

public class Person implements Serializable {
    private transient String name;//反序列化时，使name值为null
    //private  String name;//序列化时，name不能用transient修饰
    private int age;
    private char sex;

    public Person() {
        super();
    }

    public Person(String name, int age, char sex) {
        super();
        this.name = name;
        this.age = age;
        this.sex = sex;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public char getSex() {
        return sex;
    }

    public void setSex(char sex) {
        this.sex = sex;
    }

    @Override
    public String toString() {
        return "Person [name=" + name + ", age=" + age + ", sex=" + sex + "]";
    }
}

```
## Java反序列化
ObjectInputStream 对以前使用 ObjectOutputStream 写入的基本数据和对象进行反序列化

Java反序列化注意事项：**在序列的时候如果有字段不想被保存在序列化的文件中，可以使用transient修饰，然后在反序列的时候，这个字段就会反序列化不成功**
```
import java.io.FileInputStream;
import java.io.ObjectInputStream;
public class ObjectInputStreamDemo {

    public static void main(String[] args) {
        // public ObjectInputStream(InputStream in)创建从指定 InputStream 读取的
        // ObjectInputStream
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("D:/person.txt"));) {
            //public final Object readObject()从 ObjectInputStream 读取对象
            Object obj = ois.readObject();
            Person p = (Person)obj;
            System.out.println(p);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```
# 内存操作流
## ByteArrayOutputStream
此类实现了一个输出流，其中的数据被写入一个 byte 数组。缓冲区会随着数据的不断写入而自动增长。可使用 toByteArray() 和 toString() 获取数据。

需求：将一张图片转换成字节数组
```
import java.io.BufferedInputStream;
import java.io.ByteArrayOutputStream;
import java.io.FileInputStream;

public class ByteArrayOutputStreamDemo {

    public static void main(String[] args) {
        String filePath = "E:/flower.jpg";
        byte[] byteArray = fileToByteArray(filePath);
        System.out.println(byteArray.length);
    }

    /**
    * 将一个文件转换成字节数组
    * 
    * @param filePath
    *            原文件路径
    * @return 返回的字节数组，如果转换失败就返回null
    */
    public static byte[] fileToByteArray(String filePath) {
        try (
                // 先读取一张图片
                BufferedInputStream bis = new BufferedInputStream(new FileInputStream(filePath));
                // 将图片以字节的方式写入字节数组中
                ByteArrayOutputStream baos = new ByteArrayOutputStream();) {
            // 自定义容器
            byte[] buf = new byte[1024];
            // 定义一个变量用于保存每次读取的个数
            int len;
            while ((len = bis.read(buf)) != -1) {
                baos.write(buf, 0, len);
            }

            // 将字节数组取出
            byte[] byteArray = baos.toByteArray();
            return byteArray;
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
}
```
## ByteArrayInputStream
包含一个内部缓冲区，该缓冲区包含从流中读取的字节

需求：将字节数组保存成一张图片
```
import java.io.BufferedOutputStream;
import java.io.ByteArrayInputStream;
import java.io.FileOutputStream;

public class ByteArrayInputStreamDemo {
    public static void main(String[] args) {
        byte[] byteArray = ByteArrayOutputStreamDemo.fileToByteArray("E:/奥黛丽赫本.jpg");

        String savePath = "D:/test.jpg";

        byteArrayToFile(byteArray, savePath);
    }

    /**
    * 将字节数组保存到指定的文件路径中
    * 
    * @param byteArray
    *            字节数组
    * @param savePath
    *            保存文件的路径
    */
    public static void byteArrayToFile(byte[] byteArray, String savePath) {
        try (
                // 先读，从当前的字节数组里面读
                ByteArrayInputStream bais = new ByteArrayInputStream(byteArray);
                // 再写，保存到一个指定文件中
                BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream(savePath));) {
            byte[] buf = new byte[1024];
            int len;
            while ((len = bais.read(buf)) != -1) {
                bos.write(buf, 0, len);
            }
            System.out.println("保存文件成功");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
## （综合）文件再合并
要求：将文件最多分成4份，加密，再合并解密储存

步骤：
1. 从原路径“D:/小牛学堂/flower.jpg”读取文件，存储为字节数组，进行异或
2. 将字节数组分为3～4份，写入到指定目录“e:/小牛学堂”
3. 从“e:/小牛学堂”读取所有分割文件，存入内存字节数组中，再合并为完整字节数组，异或解密
4. 读取字节数组，将字节数组写入到目标路径“e:/小牛学堂/郁金香.jpg”
```
import java.io.BufferedInputStream;
import java.io.BufferedOutputStream;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;

public class SplitAndGroupFileDemo{

    public static void main(String[] args) {
        // 将指定的文件加密分割成最多4份
        splitFile("D:/小牛学堂/flower.jpg");
        groupFile("e:/小牛学堂/郁金香.jpg");
    }

    /**
    * 将分割的文件组合起来，储存到指定目标路径
    * 
    * @param saveName
    *            指定目标路径
    */
    private static void groupFile(String saveName) {
        // 将最多四部分文件读取到同一个字节数组中
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        for (int i = 0; i < 4; i++) {
            switch (i) {
            case 0:// 第1部分
                readByteArray(baos, "e:/小牛学堂/temp.001");
                break;
            case 1:// 第2部分
                readByteArray(baos, "e:/小牛学堂/temp.002");
                break;
            case 2:// 第3部分
                readByteArray(baos, "e:/小牛学堂/temp.003");
                break;
            // 判断第4个文件是否存在
            case 3:
                if (new File("e:/小牛学堂/temp.004").exists()) {
                    readByteArray(baos, "e:/小牛学堂/temp.004");
                    break;
                }
            }
        }
        byte[] byteArray = baos.toByteArray();// 定义一个byteArray接收取出的四部分文件。
        System.out.println("文件合成成功");
        // 解密
        encrypt(byteArray);
        // 将字节数组中的字节写入到目标文件中
        try (ByteArrayInputStream bais = new ByteArrayInputStream(byteArray);
                BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream(saveName));) {
            byte[] buf = new byte[1024];
            int len;
            while ((len = bais.read(buf)) != -1) {
                bos.write(buf, 0, len);
            }
            System.out.println("文件写出成功");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
    * 从指定路径读取字节到字节数组
    * 
    * @param baos
    *            字节数组
    * @param srcPath
    *            指定路径
    */
    private static void readByteArray(ByteArrayOutputStream baos, String srcPath) {
        try {
            BufferedInputStream bis = new BufferedInputStream(new FileInputStream(srcPath));
            byte[] buf = new byte[1024];
            int len;
            while ((len = bis.read(buf)) != -1) {
                baos.write(buf, 0, len);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
    * 将指定路径的文件分成3～4份
    * 
    * @param srcPath
    *            指定路径
    */
    private static void splitFile(String srcPath) {
        try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream(srcPath));
                ByteArrayOutputStream baos = new ByteArrayOutputStream();) {
            byte[] buf = new byte[1024];
            int len;
            while ((len = bis.read(buf)) != -1) {
                baos.write(buf, 0, len);
            }
            byte[] byteArray = baos.toByteArray();
            // 异或加密
            encrypt(byteArray);

            // 将字节数组最多分割成4部分保存到e:/小牛学堂
            int length = byteArray.length;
            int size = length / 3;
            BufferedOutputStream bos = null;
            for (int i = 0; i < 3; i++) {
                switch (i) {
                case 0:// 第1部分
                    bos = new BufferedOutputStream(new FileOutputStream("e:/小牛学堂/temp.001"));
                    bos.write(byteArray, 0, size);
                    break;
                case 1:// 第2部分
                    bos = new BufferedOutputStream(new FileOutputStream("e:/小牛学堂/temp.002"));
                    bos.write(byteArray, size, size);
                    break;
                case 2:// 第3部分
                    bos = new BufferedOutputStream(new FileOutputStream("e:/小牛学堂/temp.003"));
                    bos.write(byteArray, 2 * size, size);
                    break;

                }
                bos.close();
            }
            // 判断还有没有剩余的字节
            if (length > 3 * size) {
                bos = new BufferedOutputStream(new FileOutputStream("e:/小牛学堂/temp.004"));
                bos.write(byteArray, 3 * size, length - 3 * size);
                bos.close();
            }
            System.out.println("文件分割成功");

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
    * 解密或加密数组
    * 
    * @param byteArray
    *            数组
    */

    private static void encrypt(byte[] byteArray) {
        for (int i = 0; i < byteArray.length; i++) {
            byteArray[i] = (byte) (byteArray[i] ^ 25);

        }
    }

}
```