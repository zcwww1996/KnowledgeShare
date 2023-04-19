[TOC]
# 网络通信三要素
- IP地址:InetAddress
        网络中设备的标识，不易记忆，可用主机名
- 端口号
        用于标识进程的逻辑地址，不同进程的标识
- 传输协议
        常见协议：TCP，UDP
## 常用方法
```Java
import java.net.InetAddress;

/*
* InetAddress：此类表示互联网协议 (IP) 地址
*/
public class InetAddressDemo {

    public static void main(String[] args) throws Exception {
        // public static InetAddress getByName(String host)在给定主机名的情况下获取主机的 IP 地址
        InetAddress inetAddress = InetAddress.getByName("IDEA-PC");
        // public String getHostAddress()返回 IP 地址字符串（以文本表现形式）。
        String ip = inetAddress.getHostAddress();
        System.out.println(ip);
        // public String getHostName()获取此 IP 地址的主机名
        String hostName = inetAddress.getHostName();
        System.out.println(hostName);
        
        //public static InetAddress getLocalHost()返回本地主机
        InetAddress localHost = InetAddress.getLocalHost();
        System.out.println("ip="+localHost.getHostAddress()+",hostName="+localHost.getHostName());
    }

}
```
# UDP协议
DatagramSocket：此类表示用来发送和接收数据报包的套接字
客户端和服务端是两个独立的程序
客户端和服务端都有Socket

## 客户端

思路：
1. 建立udp的socket服务
2. 将要发送的数据封装成数据包，数据包中要指定地址和端口
3. 通过udp的socket服务,将数据包发送出
4. 关闭资源

```
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;
import java.util.Scanner;

public class ClientRunnable implements Runnable {
    private String ip;
    private int port;

    public ClientRunnable(String ip, int port) {
        this.ip = ip;
        this.port = port;
    }

    @Override
    public void run() {
        DatagramSocket ds = null;
        try {
            Scanner sc = new Scanner(System.in);
            // 1.创建客户端的DatagramSocket对象
            ds = new DatagramSocket();
            // 2.创建要发送的数据包并指定地址和端口
            String content;
            byte[] buf;
            InetAddress inetAddress = InetAddress.getByName(ip);
            while (!(content = sc.nextLine()).equals("886")) {
                /*
                * content = sc.nextLine(); // 如果输入的内容是886就结束录入 if ("886".equals(content)) {
                * break; }
                */
                buf = content.getBytes();
                // public void send(DatagramPacket p)从此套接字发送数据报包;
                // DatagramPacket包含的信息指示：将要发送的数据、其长度、远程主机的 IP 地址和远程主机的端口号
                DatagramPacket p = new DatagramPacket(buf, buf.length, inetAddress, port);
            // 3.将数据包发送出去
                ds.send(p);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 4.关闭资源
            if (null != ds) {
                ds.close();
            }
        }
    }

}
```
## 服务器端

思路：
1. 建立udp的socket服务，监听端口
2. 通过receive方法接收数据
3. 将收到的数据存储到数据包对象中
4. 通过数据包对象的功能来完成对接收到数据进行解析
5. 可以对资源进行关闭

```
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;

public class ServerRunnable implements Runnable {
    private int port;
    
    public ServerRunnable(int port) {
        this.port = port;
    }

    @Override
    public void run() {
        try {
            // 1.创建服务端DatagramSocket并监听指定端口
            DatagramSocket ds = new DatagramSocket(port);
            // 2.创建要保存数据的包
            byte[] buf = new byte[64 * 1024];
            DatagramPacket p = new DatagramPacket(buf, buf.length);
            InetAddress address;
            while (true) {
                ds.receive(p);// 获取数据包。阻塞式
                // 3.拆开数据包，获取数据
                byte[] data = p.getData();
                int len = p.getLength();// 实际的数据长度
                address = p.getAddress();
                String ip = address.getHostAddress();
                String hostName = address.getHostName();
                // 将读取的数据转换成字符串
                String content = new String(data, 0, len);
                System.out.println("ip=" + ip + ",主机名=" + hostName + ",内容=" + content);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
# TCP协议
## 客户端

思路：
1. 建立客服端的Socket服务，通过三次握手去连接指定地址的服务器
2. 通过Socket对象的方法获取输出流对象
3. 将数据写入流中
4. 关闭资源
### 传输文本文件
```
import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.FileReader;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.net.Socket;

//客户端文本文件，服务器输出文本文件
public class UploadTextFileClient {
    public static void main(String[] args) throws Exception {
        Socket s = new Socket("192.168.4.28", 8888);
        // 将字节输出流包装成字符缓冲输出流
        BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(s.getOutputStream()));
        // 将要写出的文件放到读取流中
        BufferedReader br = new BufferedReader(new FileReader("E:/成仙.txt"));
        // 循环读写
        char[] buf = new char[1024];
        int len;
        while ((len = br.read(buf)) != -1) {
            bw.write(buf, 0, len);
        }
        bw.flush();
        br.close();
        
        //告诉服务端一声，我没有数据写出了
        s.shutdownOutput();//禁用此套接字的输出流

        // 读取服务端的返回结果
        // 将字节读取流包装成字符缓冲读取流
        br = new BufferedReader(new InputStreamReader(s.getInputStream()));
        System.out.println(br.readLine());
        // 关闭资源
        s.close();
    }
}

```
### 传输图片文件
```
import java.io.BufferedInputStream;
import java.io.BufferedOutputStream;
import java.io.BufferedReader;
import java.io.FileInputStream;
import java.io.InputStreamReader;
import java.net.Socket;

//客户端上传图片，服务端保存起来
public class UploadImageFileClient {
    public static void main(String[] args) throws Exception {
        Socket s = new Socket("192.168.4.28", 8888);
        // 将字节输出流包装成字节缓冲输出流
        BufferedOutputStream bos = new BufferedOutputStream(s.getOutputStream());
        // 将要上传的文件封装到字节缓冲读取流
        BufferedInputStream bis = new BufferedInputStream(new FileInputStream("E:/奥黛丽赫本.jpg"));

        // 循环读写
        byte[] buf = new byte[1024];
        int len;
        while ((len = bis.read(buf)) != -1) {
            bos.write(buf, 0, len);
        }
        bos.flush();
        bis.close();

        // 告诉服务端我没数据输出了
        s.shutdownOutput();

        // 读取服务端的返回结果
        // 将字节读取流包装成字符缓冲读取流
        BufferedReader br = new BufferedReader(new InputStreamReader(s.getInputStream()));
        System.out.println(br.readLine());
        // 关闭资源
        s.close();
    }
}

```
## 服务器端

思路：
1. 建立服务端服务，监听指定的端口
2. 通过accept方法获取客户端对象
3. 获取Socket的读取流，读取客户端发送过来的数据
4. 关闭资源

### 接收文本文件
```
import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.FileWriter;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.net.ServerSocket;
import java.net.Socket;

import cn.edu360.ThreadPoolUtils;

public class UploadTextFileServer {
    public static void main(String[] args) throws Exception {
        ServerSocket ss = new ServerSocket(8888);
        // 一直接收客户端的请求
        while (true) {
            Socket s = ss.accept();
            //接到连接请求，就交给子线程来执行
            UploadTextFileRunnable uploadTextFileTask = new UploadTextFileRunnable(s);
            ThreadPoolUtils.execute(uploadTextFileTask);
            // 将字节读取流包装成字符缓冲读取流
            BufferedReader br = new BufferedReader(new InputStreamReader(s.getInputStream()));
            // 将文本输出到一个文件中
            BufferedWriter bw = new BufferedWriter(new FileWriter("D:/" + System.currentTimeMillis() + ".txt"));
            // 循环读写
            char[] buf = new char[1024];
            int len;
            while ((len = br.read(buf)) != -1) {
                bw.write(buf, 0, len);
            }
            // 自己创建的流需要关闭
            bw.close();

            // 将结果返回给客户端
            System.out.println("上传文件成功");
            // 将字节输出流包装成字符缓冲输出流
            bw = new BufferedWriter(new OutputStreamWriter(s.getOutputStream()));
            bw.write("上传文件成功");
            bw.newLine();
            bw.flush();
            s.close();
        }
    }
}
```
### 接收图片文件
- 使用子线程，提供效率
```
import java.io.BufferedInputStream;
import java.io.BufferedOutputStream;
import java.io.BufferedWriter;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.OutputStreamWriter;
import java.net.Socket;

public class UploadImageFileRunnable implements Runnable {
    private Socket s;

    public UploadImageFileRunnable(Socket s) {
        this.s = s;
    }

    @Override
    public void run() {
        try {
            // 将字节读取流包装成字节缓冲读取流
            BufferedInputStream bis = new BufferedInputStream(s.getInputStream());
            // 将字节数据保存到指定的文件中
            BufferedOutputStream bos = new BufferedOutputStream(
                    new FileOutputStream("D:/" + System.currentTimeMillis() + ".jpg"));

            byte[] buf = new byte[1024];
            int len;
            while ((len = bis.read(buf)) != -1) {
                bos.write(buf, 0, len);
            }
            // 自己创建的流需要关闭
            bos.close();

            // 将最终的结果输出给客户端
            System.out.println("上传图片成功");
            // 将字节输出流包装成字符缓冲输出流
            BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(s.getOutputStream()));
            bw.write("上传图片成功");
            bw.newLine();
            bw.flush();
        } catch (Exception e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        } finally {
            if (null != s) {
                // 关闭资源
                try {
                    s.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```