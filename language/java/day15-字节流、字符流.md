[TOC]
# IO流：

*   I：In
*   O：Out

## 按照流的数据类型：

### 字节流：操作任何数据

#### 按照流的方向来分：

*   字节输出流基类：OutputStream

*   字节读取流基类：InputStream

### 字符流：只能操作文本数据

#### 按照流的方向来分：

*   字符输出流基类：Writer

*   字符读取流基类：Reader

**字节流和字符流的子类都是以它们父类的后缀名结尾的**

# 应用

字节流和字符流应用场景

*   BufferedReader：从字符输入流中读取文本，缓冲各个字符，从而实现字符、数组和行的高效读取

public String readLine()读取一个文本行

*   BufferedWriter：将文本写入字符输出流，缓冲各个字符，从而提供单个字符、数组和字符串的高效写入

public void newLine()写入一个**行分隔符**

_拷贝的文件必须是一个标准文件_

*   当你拷贝文本文件时：使用字符缓冲流，拷贝的时候读取字符数组和写出字符数组

*   当你拷贝的不是文本文件时：使用字节缓冲流

*   如果你不知道要拷贝的文件类型，那么就用字节缓冲流

# 问题

## 创建流做了哪些事情？

见画的图

## 如何实现数据的追加写入？

在创建输出流的时候将第二个参数值传入true。 
new FileOutputStream(pathname, true)

字节流写出代码第17行

## 如何实现数据的换行？

在每一行数据写完之后再写入一个换行分隔符

## 如何获取当前工作空间编码？

String name = Charset.defaultCharset().name();

字节流写出代码第11行

## 如何实现关闭流？

见代码
```java
// JDK1.7新特性：自动关闭流
    try(创建流的代码){

         该怎么写还怎么写

    }catch(....){

        ...

    }
```

# 代码

## 字节流写出

使用JDK1.7新特性关流

```java
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.OutputStream;
 // 文件输出流是用于将数据写入 File

 // 需求：向“D:/test.txt”文件中写入"HelloWorld"

public class FileOutputStreamDemo {
    public static void main(String[] args) {

        // 获取当前工作空间的编码

        // String name = Charset.defaultCharset().name();

        // System.out.println(name);//UTF-8

        String pathname = "D:/test.txt";

        // 当fos使用完之后就会自动关闭

        try (FileOutputStream fos = new FileOutputStream(pathname, true);) {

            // public FileOutputStream(String name)创建一个向具有指定名称的文件中写入数据的输出文件流

            byte[] bytes = "HelloWorld\r\n".getBytes();

            // public void write(int b)将指定字节写入此文件输出流

            // fos.write(bytes[0]);

            // public void write(byte[] b)将 b.length 个字节从指定 byte 数组写入此文件输出流中

            fos.write(bytes);

            // public void write(byte[] b,int off,int len)
            //将指定 byte 数组中从偏移量 off 开始的 len 个字节写入此文件输出流
            // fos.write(bytes, 5, 5);

            System.out.println("写入成功");

        } catch (Exception e) {

            e.printStackTrace();
        }
    }
}
```

## 字节流读入

使用JDK1.7新特性关流

```java
import java.io.FileInputStream;

import java.io.IOException;

import java.io.InputStream;
//从“D:/test.txt”中读取字节



public class FileInputStreamDemo {

    public static void main(String[] args) {



        // FileInputStream 从文件系统中的某个文件中获得输入字节



        // public FileInputStream(String name)通过打开一个到实际文件的连接来创建一个FileInputStream，该文件通过文件系统中的路径名 name 指定


        try (FileInputStream fis = new FileInputStream("D:/test.txt");) {


            // public int read(byte[] b)从此输入流中将最多 b.length 个字节的数据读入一个 byte

            // 数组中;读入缓冲区的字节总数，如果因为已经到达文件末尾而没有更多的数据，则返回 -1

            byte[] buf = new byte[1024];

            // fis.skip(1);// 跳过第一个字节

            // public long skip(long n)从输入流中跳过并丢弃 n 个字节的数据

            int len = fis.read(buf);

            // public int read(byte[] b,int off,int len)从此输入流中将最多 len 个字节的数据读入一个

            String result = new String(buf, 0, len);

            // 将读取的字节转换成字符串

            System.out.println(result);



        } catch (Exception e) {

            e.printStackTrace();

        }

    }

}
```

## 使用字节缓冲流拷贝标准文件

*   使用JDK1.7新特性关流

*   使用缓冲区

### flush什么时候用？

*   或者当缓冲区满的时候被动刷新出去；
*   当有一部分数据很着急需要刷新出去的时候，使用flush；

*   如果写出的数据不是那么着急使用，那么可以在关闭流的时候（JVM自动调用flush方法）直接刷新出去；

```java
import java.io.BufferedInputStream;

import java.io.BufferedOutputStream;

import java.io.FileInputStream;

import java.io.FileOutputStream;
 //把E:/奥黛丽赫本.jpg内容复制到当D:/mn.jpg中



public class FileCopyDeom {
    public static void main(String[] args) {

        try (

                // 先读

                FileInputStream fis = new FileInputStream("E:/奥黛丽赫本.jpg");               
                /*public BufferedInputStream(InputStream in)创建一个BufferedInputStream 并保存其参数，即输入流 in，以便将来使用。
                    创建一个内部缓冲区数组并将其存储在buf 中*/
                BufferedInputStream bis = new BufferedInputStream(fis);



                // 后写

                FileOutputStream fos = new FileOutputStream("D:/mn.jpg");

                // public BufferedOutputStream(OutputStream out)创建一个新的缓冲输出流，以将数据写入指定的底层输出流。

               BufferedOutputStream bos = new BufferedOutputStream(fos);) {


            // 自定义缓冲区

            byte[] buf = new byte[1024];

            // 自定义一个变量用于保存每次读取的长度

            int len;

            // 循环读写

            while ((len = bis.read(buf)) != -1) {

                //如果不等于-1表明还有数据可以读取

                bos.write(buf, 0, len);

                 // 读多少就写出多少//
            }

            System.out.println("拷贝成功");

        } catch (Exception e) {

            e.printStackTrace();

        }

    }

}
```

## 字节流拷贝数据方法（封装）

*   copyFile 使用普通字节流拷贝文件
*   copyFile2 使用缓冲字节流拷贝文件

```java
import java.io.BufferedInputStream;
import java.io.BufferedOutputStream;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
public class IOUtils {
    /**
     * 使用字节缓冲流拷贝文件
     * 
     * @param srcPath
     *            原文件路径
     * @param destPath
     *            目标文件路径
     */
    public static void copyFile2(String srcPath, String destPath) {
        try (
                FileInputStream fis = new FileInputStream(srcPath);
                BufferedInputStream bis = new BufferedInputStream(fis);
                FileOutputStream fos = new FileOutputStream(destPath);
                BufferedOutputStream bos = new BufferedOutputStream(fos);)
        {
            byte[] buf = new byte[1024];
            int len;
            while ((len = bis.read(buf)) != -1) {
                bos.write(buf, 0, len);
            }
            System.out.println("拷贝成功");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    /**
     * 使用普通字节流拷贝文件
     * 
     * @param srcPath
     *            原文件路径
     * @param destPath
     *            目标文件路径
     */
    public static void copyFile(String srcPath, String destPath) {
        try (
                FileInputStream fis = new FileInputStream(srcPath);
                FileOutputStream fos = new FileOutputStream(destPath);) {

            byte[] buf = new byte[1024];
            int len;
            while ((len = fis.read(buf)) != -1) {
                fos.write(buf, 0, len);
            }
            System.out.println("拷贝成功");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    /**
     * 关闭字节读取流
     * 
     * @param fis
     *            字节读取流
     */
    public static void closeIn(InputStream fis) {
        if (null != fis) {
            try {
                fis.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    /**
     * 关闭字节输出流
     * 
     * @param fos
     *            字节输出流
     */
    public static void closeOut(OutputStream fos) {
        if (null != fos) {
            try {
                fos.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

## 字符流写出

使用JDK1.7新特性关流

```java
import java.io.FileOutputStream;

import java.io.OutputStreamWriter;
/*
 * 编码：将看的懂的变成看不懂的
 * 解码：将看不懂的变成看的懂的
 * 
 * 字符流=字节流+编码表
 * 编码表：就是字符对应数值的一张表
 * 
 * OutputStreamWriter 是字符流通向字节流的桥梁
 * OutputStreamWriter=字节输出流+编码表
 */
public class OutputStreamWriterDemo {
    public static void main(String[] args) {
        try (
                FileOutputStream fos = new FileOutputStream("D:/test.txt");
                // public OutputStreamWriter(OutputStream out,String charsetName)创建使用指定字符集的
                // OutputStreamWriter
                OutputStreamWriter osw = new OutputStreamWriter(fos, "utf-8");
                ) {
            // public void write(String str)写入字符串
            osw.write("HelloWorld");
            // public void write(char[] cbuf)写入字符数组
            // char[] cbuf = {'中','国','你','好','\r','\n'};
            // public abstract void write(char[] cbuf,int off,int len)写入字符数组的某一部分
            // osw.write(cbuf,0,6);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## 字符流读入

使用JDK1.7新特性关流

```java
package cn.edu360;
import java.io.FileInputStream;
import java.io.InputStreamReader;
/*
 * InputStreamReader 是字节流通向字符流的桥梁
 * InputStreamReader=字节读取流+编码表
 */
public class InputStreamReaderDemo {
    public static void main(String[] args) {
        try (
                FileInputStream fis = new FileInputStream("D:/test.txt");
                // 创建使用指定字符集的 InputStreamReader
                // public InputStreamReader(InputStream in,String charsetName)
                InputStreamReader isr = new InputStreamReader(fis, "utf-8");
                ) {
            char[] cbuf = new char[1024];
            // public int read(char[] cbuf)将字符读入数组；读取的字符数，如果已到达流的末尾，则返回 -1
            int len = isr.read(cbuf);
            // 将字符数组转换成字符串
            String result = new String(cbuf, 0, len);
            System.out.println(result);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
## 字符缓冲流读入

```java
import java.io.*;

class BufferReaderDemo {
    public static void main(String[] args) {
        try {
            //读取文件，并且以utf-8的形式写出去
            String line = null;
            File file = new File("d://desktop//test.txt");
            FileInputStream fis = new FileInputStream(file);
            BufferedReader br = new BufferedReader(new InputStreamReader(fis, "utf8"));
            while ((line = br.readLine()) != null) {
                System.out.println(line);
            }
            br.close();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## 使用字符缓冲流拷贝文本文件

*   使用JDK1.7新特性关流

*   使用缓冲区

```java
import java.io.FileReader;
import java.io.FileWriter;
/*
 * FileReader：用来读取字符文件的便捷类。此类的构造方法假定默认字符编码和默认字节缓冲区大小都是适当的。
 *             要自己指定这些值，可以先在 FileInputStream 上构造一个 InputStreamReader。 
 * 
 * FileWriter：用来写入字符文件的便捷类。此类的构造方法假定默认字符编码和默认字节缓冲区大小都是可接受的。
 *             要自己指定这些值，可以先在 FileOutputStream 上构造一个 OutputStreamWriter
 */
public class CopyFileDemo {
    public static void main(String[] args) {
        try(
                //public FileReader(String fileName)在给定从中读取数据的文件名的情况下创建一个新 FileReader
                FileReader fr = new FileReader("E:/a.txt");
                //public FileWriter(String fileName)根据给定的文件名构造一个 FileWriter 对象
                FileWriter fw = new FileWriter("D:/copy.txt");
                ) {
            //自定义容器
            char[] buf = new char[1024];
            //定义一个变量用于保存每次读取的字符个数
            int len;
            //循环读写
            while((len=fr.read(buf))!=-1){
                fw.write(buf, 0, len);
            }
            System.out.println("拷贝文本文件成功");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## 字符流拷贝文本文件方法（封装）

*   copyFile3 使用普通字符流拷贝文件
*   copyFile4 使用缓冲字符流拷贝文件

```java
import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.FileReader;
import java.io.FileWriter;


public class IOUtils2 {

    /**
     * 使用字符缓冲流拷贝文本文件
     * 
     * @param srcPath
     *            原文件路径
     * @param destPath
     *            目标文件路径
     */
    public static void copyFile4(String srcPath, String destPath) {
        try (
                FileReader fr = new FileReader(srcPath);
                BufferedReader br = new BufferedReader(fr);
                FileWriter fw = new FileWriter(destPath);
                BufferedWriter bw = new BufferedWriter(fw);
                ) {
            char[] buf = new char[1024];
            int len;
            while ((len = br.read(buf)) != -1) {
                bw.write(buf, 0, len);
            }
            System.out.println("拷贝文本文件成功");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    /**
     * 使用普通字符流拷贝文件文件
     * 
     * @param srcPath
     *            原文件路径
     * @param destPath
     *            目标文件路径
     */
    public static void copyFile3(String srcPath, String destPath) {
        try (
                FileReader fr = new FileReader(srcPath); 
                FileWriter fw = new FileWriter(destPath);
                ) {
            char[] buf = new char[1024];
            int len;
            while ((len = fr.read(buf)) != -1) {
                fw.write(buf, 0, len);
            }
            System.out.println("拷贝文本文件成功");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```