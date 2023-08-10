[TOC]

# 1. resultSet转bean

mysql的结果集ResultSet存入bean中是十分麻烦的，需要一个工具类来简化，该工具类利用Java反射实现


## 1.1 工具类
```Java
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Timestamp;
import java.util.regex.Matcher;
import java.util.regex.Pattern;


public class RsToBeanUtil {

    /**
     * 类名将查询结果resultSet封装成对象
     * 属性名必须严格遵守驼峰命名，该类会将属性驼峰转为数据库_ 去与数据库字段进行匹配
     *
     * @param rs 查询结果集
     * @param t  要封装的对象
     * @return 返回封装好的对象
     */
    public static <T> T resultToBean(ResultSet rs, Class<T> t) {
        T result = null; // 封装数据完返回的Bean对象

        try {
            // 无参构造 创建bean对象
            result = t.newInstance();
            // 获得bean的属性数组
            Field[] fields = t.getDeclaredFields();


            for (Field field : fields) {
                // 取出属性
                // 获取类属性名
                String fieldName = field.getName();

                // 属性名转数据库字段，驼峰转下划线
                Pattern humpPattern = Pattern.compile("[A-Z]");
                Matcher matcher = humpPattern.matcher(fieldName);
                StringBuffer sb = new StringBuffer();
                while (matcher.find()) {
                    matcher.appendReplacement(sb, "_" + matcher.group(0).toLowerCase());
                }
                matcher.appendTail(sb);
                String sqlName = sb.toString();

                // 用属性名得出set方法，set+将首字母大写
                String setMethodName = "set" + fieldName.substring(0, 1).toUpperCase() + fieldName.substring(1);

                // 获取字段类型
                String type = field.getType().toString();

                // 通过调用set方法进行属性注入
                switch (type) {
                    case "class java.lang.Integer": {
                        // 类型匹配 则获取对应属性的set方法    方法名，类型
                        Method method = t.getMethod(setMethodName, Integer.class);
                        // 执行set方法    被执行的对象，需要注入的数据
                        method.invoke(result, rs.getInt(sqlName));
                        break;
                    }
                    case "class java.lang.String": {
                        Method method = t.getMethod(setMethodName, String.class);
                        method.invoke(result, rs.getString(sqlName));
                        break;
                    }
                    case "class java.lang.Double": {
                        Method method = t.getMethod(setMethodName, Double.class);
                        method.invoke(result, rs.getDouble(sqlName));
                        break;
                    }
                    case "class java.sql.Timestamp": {
                        Method method = t.getMethod(setMethodName, Timestamp.class);
                        method.invoke(result, rs.getTimestamp(sqlName));
                        break;
                    }
                    default:
                        throw new RuntimeException("类型异常");
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

        // 返回该对象
        return result;
    }

```

## 1.2 工具类的使用

**使用工具类前**

```Java
while (rs.next()) {
	Cost cost = new Cost();
    cost.setCostId(rs.getInt("cost_id"));
    cost.setName(rs.getString("name"));
    cost.setBaseDuration(rs.getInt("base_duration"));
    cost.setBaseCost(rs.getDouble("base_cost"));
	cost.setUnitCost(rs.getDouble("unit_cost"));
	cost.setStatus(rs.getString("status"));
    cost.setDescr(rs.getString("descr"));
    cost.setCreatime(rs.getTimestamp("creatime"));
    cost.setStartime(rs.getTimestamp("startime"));
    cost.setCostType(rs.getString("cost_type"));
    list.add(cost);
}

```

**使用工具类后**

```Java
Class<Cost> costClass = Cost.class;
while (rs.next()) {
	Cost cost = RsToBeanUtil.resultToBean(rs, costClass);
	list.add(cost);
}

```
