---
title: 关于jar
categories: java
tags:
- tools
---

## jar介绍
  使用JAR（Java Archive）的文件格式能够把多个文件打包为一个文件。一个jar文件包含class文件，与程序相关的辅助性资源文件。
  使用jar有以下优点：
  - 安全性：对jar文件进行数字签名，方便认识签名的用户对软件进行授权。
  - 缩短下载时间：所有的文件都在一个jar文件中，一次http就能把全部文件都下下来。
  - 压缩：jar文件格式压缩了文件的大小，节省存储空间。
  - 扩展性：把一个功能甚至一个软件打包成一个jar包，作为扩展的包加入到其他应用中。
  - 密封性：package中包含的所有class都要在jar中，对package强制版本一致性。
  - 版本：jar文件包含了供应商和版本信息。
  - 可移植性：处理JAR文件的机制是Java平台核心API标准的一部分。

## jar的简单用法
  JDK提供了Java Archive Tool，使用jar命令就能调用这个工具来打jar包。

  操作                                     | 命令
  ---------------------------------------- | ---------------------------
  创建jar文件                              | jar cf jar文件 输入文件
  查看jar文件内容                          | jar tf jar文件
  提取jar文件内容                          | tar xf jar文件
  从jar中提取特定文件                      | tar xf jar文件 提取文件名
  运行jar文件（需要Main-class的Manifest头）| java -jar jar文件
  将jar作为applet调用                      | fjdk<applet code=AppletClassName.class<br>　　archive="JarFileName.jar"<br>　　width=width height=heightniam>niam</applet>:w


## jar的Manifest
  Manifest文件包含了jar文件中打包的文件的信息，使得jar文件有电子签名，版本控制，包密封以及其他功能。
    
### 默认的Manifest文件
  当创建一个jar文件时会自动生成一个默认的manifest文件，且jar文件中只能有一个manifest，路径是META-INF/MANIFEST.MF。
  默认的manifest有以下内容：
  ```
  Manifest-Version: 1.0
  Created-By: 1.7.0_06 (Oracle Corporation)
  ```
  manifest中的条目采用"header: value"的形式，manifest中要记录什么信息取决于你想要使用jar的哪些功能。

### 修改Manifest文件
  要开启jar文件的某个功能，需要把相应的header加入manifest文件中。先写好一个文本文件，包含要添加的内容，然后使用jar命令的m选项就能把文本文件的内容添加到manifest中。
  ```
  jar cfm jar文件 文本文件 输入文件
  ```
  解释一下这条命令：
  - c选项表示创建一个jar文件
  - m选项表示你想把文本文件中的内容合并到manifest文件中
  - f选项表示把命令的输出保存到jar文件中，而不是输出到标准输出


  **注意：m和f选项的顺序必须和后面jar文件和文本文件的顺序一致。文本文件必须以新行或回车结尾，否则无法正确解析最后一行。**

### 设置应用程序的入口点
  当把应用程序打包成jar文件时，需要指明jar文件中的哪个class是应用程序的入口点。Main-Class头用来提供这个信息。
  ```
  Main-Class: classname
  ```
  所谓的入口点就是有public static void main(String[] args)函数的class。设置了入口点以后，就可以用
  ```
  java -jar jar文件
  ```
  运行程序，Main-Class指明的class的main函数会被执行。
  也可以直接在命令中指定入口点，使用jar命令的e选项：

  ```
  jar cfe jar文件 入口点class名 输入文件
  ```


### 使用Class-Path添加class
  有时一个jar包中的文件可能会引用其他jar包中的文件，这时需要使用Class-Path头来指明jar包。
  ```
  Class-Path: jar文件1 jar文件2 目录名/jar文件3
  ```
  使用Class-Path的好处是可以避免在java命令中指明很长的-classpath。

  **注意：jar包可能是嵌套的，一个jar文件可能包含jar文件，要将被包含的jar包中的class添加到classpath时，要使用定制的代码来装载这些类，而不能用Class-Path。**

### 设置package版本信息
  有关版本信息的manifest header如下：

  header                  |   Definition 
  ----------------------- | ---------------------------------
  Name                    | The name of the specification.
  Specification-Title     | The title of the specification.
  Sepcrfication-Version   | The version of the specification.
  Specrfication-Vendor    | The vendor of the specification.
  Implementation-Title    | The title of the implementation.
  Implementation-Version  | The build number of the implementation.
  Implementation-Vendor   | The vendor of the implementation.


### 在jar文件中密封package
  可在jar文件中密封package，密封的意思是package中的所有class文件必须被打包到同一个jar文件中。为了保证class版本的一致性，会有这种密封package的需求。
  在manifest中指明密封的通用格式为：
  ```
  Name: myCompany/myPackage/
  Sealed: true
  ```
 **注意：package名要以"/"作为结束。** 

  可以密封jar文件以确保jar文件中的所有package都被密封,在package上能够重写这个设置。密封jar的命令不需要指明Name，使用```Sealed: true```就可以。

**参见：[java jar 文档](https://docs.oracle.com/javase/tutorial/deployment/jar/index.html)**
