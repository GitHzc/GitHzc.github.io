---
title: Protocol Buffers
tags: tools
---

# protocol buffers是什么
- protocol buffers是一种灵活，可扩展，自动的，用来序列化结构化数据的机制。
- 相比XML，protocol buffers更小型，快速和简单。
- 定义好数据的结构，使用生成的代码来从数据流中读写数据。
- 数据的结构有更新时，不会影响到使用旧数据结构的程序。
<!-- more -->
# protocol buffers如何运作
1. 在.proto文件中message类型指明要序列化的数据的结构。message是信息的逻辑记录，包含一系列name-value对。
```
message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    required string number = 1;
    optional PhoneType type = 2 [default = HOME];
  }

  repeated PhoneNumber phone = 4;
}
```
每个message类型都有一个或多个唯一的带编号字段，每一个字段有name和value，value的类型可以是数字（整数或浮点数），布尔值，字符串，raw bytes，甚至是其他的message类型。字段前可以使用optional、required、repeated等修饰。

2. 定义好message之后，运行protocol buffers complier，将生成数据的访问类。这些类提供了访问类中字段的简单方法，像name()和set_name()，同时也提供了方法来将数据序列化到raw byte，以及从raw byte反序列化为数据。如果使用的语言的c++，在上面的例子中运行complier将会生成Person类，能像下面这样来使用Person：
```
// serial data
Person person;
person.set_name("John Doe");
person.set_id(1234);
person.set_email("jdoe@example.com");
fstream output("myfile", ios::out | ios::binary);
person.SerializeToOstream(&output);
// parse data
fstream input("myfile", ios::in | ios::binary);
Person person;
person.ParseFromIstream(&input);
cout << "Name: " << person.name() << endl;
cout << "E-mail: " << person.email() << endl;
```
3. 在message中添加新字段不会破坏后向兼容性，在解析的时候使用旧message的程序会忽略新加入的字段。

# 为什么不使用XML
相比XML，protocol buffers有以下几个优点：
- 更简单
- 比XML小3-10倍
- 比XML快20-100倍
- 更明确
- 能生成易于编程使用的数据访问类

例如，对包含name和email字段的person进行建模时，XML是这样写的：
```
<person>
  <name>John Doe</name>
  <email>jdoe@example.com</email>
</person>
```
对应的protocol buffers message是这样写的：
```
# Textual representation of a protocol buffer.
# This is *not* the binary format used on the wire.
person {
  name: "John Doe"
  email: "jdoe@example.com"
}
```
当编码为二进制格式时，protocol buffers使用28个字节，解析需要100-200纳秒。XML至少需要69个字节，解析需要5000-10000纳秒。

protocol buffers对数据的操作更加容易：
```
cout << "Name: " << person.name() << endl;
cout << "E-mail: " << person.email() << endl;
```
XML的操作方式：
```
cout << "Name: "
     << person.getElementsByTagName("name")->item(0)->innerText()
     << endl;
cout << "E-mail: "
     << person.getElementsByTagName("email")->item(0)->innerText()
     << endl;
```

但是，protocol buffers并不总是优于XML。在对带有标记的文本文件（如HTML）进行建模时，XML要好于protocol buffers，因为protocol buffers不能把文本和结构交织在一起。

参见：[Developer Guide](https://developers.google.com/protocol-buffers/docs/overview)
