[TOC]

# 1. 自定义jar包

自定义jar包，里面包含主类GetGeoHashRangeJ，两个方法invoke1、invoke2
```java
import com.smartsteps.www.util.GridDeal;

public class GetGeoHashRangeJ {
    public static String[] invoke1(String[] arr) {
        String lat = arr[0];
        String lon = arr[1];
        String radius = arr[2];
        double latNum = Double.parseDouble(lat);
        double lonNum = Double.parseDouble(lon);
        int radiusNum = Integer.parseInt(radius) * 1000;
        String[] circleGeohash = GridDeal.getCircleGeohash(latNum, lonNum, radiusNum);
        return circleGeohash;
    }

    public static String[] invoke2(String lat, String lon, String radius) {
        double latNum = Double.parseDouble(lat);
        double lonNum = Double.parseDouble(lon);
        int radiusNum = Integer.parseInt(radius) * 1000;
        String[] circleGeohash = GridDeal.getCircleGeohash(latNum, lonNum, radiusNum);
        return circleGeohash;
    }
}
```

# 2. 动态调用jar包里面的方法

```java
import java.lang.reflect.Method;
import java.net.URL;
import java.net.URLClassLoader;
import java.util.Arrays;

public class jarMain {

    public static void main(String[] args) throws Exception {
        // Jar包所在的绝对路径，过url加载器加载，这里是本地路径，需要file前缀，也可以是网络路径，如：http://www.xxx.com/xxx.jar
        String softPath = "file:D:\\Code\\ideaCode\\process-xaa-requests\\call-spark\\target\\call-spark-0.0.1-SNAPSHOT.jar";
        // 创建URL数组，包含Jar包的路径
        URL[] urls = new URL[]{new URL(softPath)};

        // 创建URLClassLoader，用于加载Jar包
        URLClassLoader classLoader = new URLClassLoader(urls);

        // 加载主类GetGeoHashRangeJ
        Class<?> clazz = Class.forName("com.smartsteps.www.func.GetGeoHashRangeJ", true, classLoader);

        // 创建一个实例对象(如果该方法是静态的，这一步可以省略）
        Object instance = clazz.newInstance();

        /*
            GetGeoHashRangeJ.invoke1参数是String[] arr
         */
        // 获取invoke方法
        Method invokeMethod1 = clazz.getMethod("invoke1", String[].class);
        String[] input = new String[]{"39.905536", "116.401915", "1"};
        // GetGeoHashRangeJ.invoke参数是String[]，所以要把String[]转成Object
        String[] result1 = (String[]) invokeMethod1.invoke(instance, (Object) input);
        System.out.println("invoke1 result");
        System.out.println(Arrays.toString(result1));

        System.out.println("------------------------------------------------------");

        /*
           GetGeoHashRangeJ.invoke2参数是String lat, String lon, String radius
          */
        // 获取invoke方法
        Method invokeMethod2 = clazz.getMethod("invoke2", String.class, String.class, String.class);
        String[] result2 = (String[]) invokeMethod2.invoke(instance, "39.905536", "116.401915", "1");
        System.out.println("invoke2 result");
        System.out.println(Arrays.toString(result2));

    }
}
```
