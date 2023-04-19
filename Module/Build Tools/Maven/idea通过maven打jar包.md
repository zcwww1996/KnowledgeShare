[TOC]
# 1. maven打jar包
## 1.1 打包过程
打jar包插件，Maven projects -> Lifecycle -> package

target目录生成两个jar包:
1. original-XXX.jar-1.0-SNAPSHOT.jar（小包，不含依赖包）、
2. XXX.jar-1.0-SNAPSHOT.jar（大包，含第三方依赖包）


[![QOWWmF.png](https://s2.ax1x.com/2019/12/20/QOWWmF.png)](https://upload.cc/i1/2019/12/20/Hr1JpE.png)
[![QOWfw4.png](https://s2.ax1x.com/2019/12/20/QOWfw4.png)](https://upload.cc/i1/2019/12/20/IDFkB6.png)

### 1.1.1 常用操作区别

**mvn clean compile**<br> 依次执行了clean、resources、compile等3个阶段，成功之后，即可在<根目录>/target找到编译出来的class文件

mvn compile 编译source code,编译范围**只包含main下的内容**，如果也想编译test，可以单独执行mvn test-compile

**mvn clean package**<br>
依次执行了clean、resources、compile、testResources、testCompile、test、jar(打包)等７个阶段。

mvn package 编译source code并按pom的定义打包，**编译范围包含mian和test**

命令完成了项目编译、单元测试、打包功能，执行完成之后，即可在target文件夹内找到jar文件。但没有把打好的可执行jar包（war包或其它形式的包）布署到本地maven仓库和远程maven私服仓库

**mvn clean install**<br>
依次执行了clean、resources、compile、testResources、testCompile、test、jar(打包)、install等8个阶段。

mvn package 编译source code并按pom的定义打包，**编译范围包含mian和test**

命令完成了项目编译、单元测试、打包功能，同时把打好的可执行jar包（war包或其它形式的包）**布署到本地maven仓库**，但没有布署到远程maven私服仓库

**mvn clean deploy**<br>
依次执行了clean、resources、compile、testResources、testCompile、test、jar(打包)、install、deploy等９个阶段。

命令完成了项目编译、单元测试、打包功能，同时把打好的可执行jar包（war包或其它形式的包）布署到本地maven仓库和远程maven私服仓库


## 1.2 打包插件
```xml
<build>
        <pluginManagement>
            <plugins>
                <!-- 编译scala的插件 -->
                <plugin>
                    <groupId>net.alchim31.maven</groupId>
                    <artifactId>scala-maven-plugin</artifactId>
                    <version>3.2.2</version>
                </plugin>
                <!-- 编译java的插件 -->
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.5.1</version>
                </plugin>
            </plugins>
        </pluginManagement>
        <plugins>
            <plugin>
                <groupId>net.alchim31.maven</groupId>
                <artifactId>scala-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>scala-compile-first</id>
                        <phase>process-resources</phase>
                        <goals>
                            <goal>add-source</goal>
                            <goal>compile</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>scala-test-compile</id>
                        <phase>process-test-resources</phase>
                        <goals>
                            <goal>testCompile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
 
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <executions>
                    <execution>
                        <phase>compile</phase>
                        <goals>
                            <goal>compile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
 
 
            <!-- 打jar插件 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.4.3</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
</build>
```
