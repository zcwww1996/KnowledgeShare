[TOC]
# File
## File路径表示形式
```
import java.io.File;

/*
* File：文件和目录路径名的抽象表示形式。 
*/
public class FileDemo {

    public static void main(String[] args) {
        String pathname = "E:/aa/bb/奥黛丽赫本.jpg";
        
        //★★★public File(String pathname)通过将给定路径名字符串转换为抽象路径名来创建一个新 File 实例
        File file = new File(pathname);
        
        // public File(String parent,String child)根据 parent 路径名字符串和 child 路径名字符串创建一个新 File 实例
        String parent = "E:/aa/bb/";
        String child = "奥黛丽赫本.jpg";
        file = new File(parent,child);
        
        // public File(File parent,String child)根据 parent 抽象路径名和 child 路径名字符串创建一个新 File 实例
        File parentFile = new File(parent);
        file = new File(parentFile,child);

    }

}

```
## File的基本功能
```
import java.io.File;
import java.io.IOException;

public class FileDemo2 {

    public static void main(String[] args) throws Exception {
        File file = new File("D:/test.txt");
        
        // 创建功能
        // public boolean createNewFile()当且仅当不存在具有此抽象路径名指定名称的文件时，不可分地创建一个新的空文件
        System.out.println(file.createNewFile());
        
        // public boolean mkdir()创建此抽象路径名指定的目录
        //file = new File("D:/test");
        //System.out.println(file.mkdir());
        
        // public boolean mkdirs()创建此抽象路径名指定的目录，包括所有必需但不存在的父目录
        //file = new File("D:/aa");
        //System.out.println(file.mkdirs());
        
        // 删除功能
        // public boolean delete()删除此抽象路径名表示的文件或目录；只能删除空文件夹或者标准文件
        //System.out.println(file.delete());
        
        // 重命名功能
        // public boolean renameTo(File dest)重新命名此抽象路径名表示的文件，还具有剪切功能
        File dest = new File("E:/haha.txt");
        file.renameTo(dest);
    }

}

```
## File判断功能
```
import java.io.File;

public class FileDemo3 {

    public static void main(String[] args) {
        File file = new File("E:/奥黛丽赫本.jpg");
        
        // 判断功能
        // public boolean isDirectory()判断此文件是否是一个文件夹
        System.out.println(file.isDirectory());//false
        
        // public boolean isFile()判断此文件是否是一个标准文件
        System.out.println(file.isFile());//true
        
        // public boolean exists()判断此文件是否存在
        System.out.println(file.exists());//true
        
        // public boolean canRead()判断此文件是否可读
        System.out.println(file.canRead());//true
        
        // public boolean canWrite()判断此文件是否可写
        System.out.println(file.canWrite());//true
        
        // public boolean isHidden()判断此文件是否是隐藏的
        file = new File("E:/haha.txt");
        System.out.println(file.isHidden());//true
    }

}

```
## File的获取功能
```
import java.io.File;
import java.text.SimpleDateFormat;
import java.util.Date;

public class FileDemo4 {

    public static void main(String[] args) {
        File file = new File("./src/cn/edu360/FileDemo2.java");
        
        // 基本获取功能
        // public String getAbsolutePath()获取全路径名，开发中用这个★★★
        System.out.println(file.getAbsolutePath());
        
        // public String getPath()获取相对路径名
        System.out.println(file.getPath());
        
        // public String getName()获取文件的名字
        System.out.println(file.getName());//FileDemo2.java
        
        // public long length()获取文件的字节长度
        System.out.println(file.length());//1147
        
        // public long lastModified()获取最后修改的时间
        long time = file.lastModified();
        System.out.println(new SimpleDateFormat().format(new Date(time)));
        
        // 高级获取功能
        // public String[] list()返回一个字符串数组，这些字符串指定此抽象路径名表示的目录中的文件和目录
        file = new File("E:/aa");
        String[] list = file.list();
        for (String path : list) {
            System.out.println(path);
        }
        
        System.out.println("---------------------------------");
        
        // public File[] listFiles()返回一个抽象路径名数组，这些路径名表示此抽象路径名表示的目录中的文件
        File[] files = file.listFiles();
        for (File f : files) {
            System.out.println(f);
        }
    }

}

```