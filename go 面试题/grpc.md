# grpc应用详解与实例剖析

[gRPC ](https://grpc.io/)是一个高性能、通用的开源RPC框架，基于HTTP/2协议标准和Protobuf序列化协议开发，支持众多的开发语言。



# 概述

在gRPC框架中，客户端可以像调用本地对象一样直接调用位于不同机器的服务端方法，如此我们就可以非常方便的创建一些分布式的应用服务。

在服务端，我们实现了所定义的服务和可供远程调用的方法，运行一个gRPC server来处理客户端的请求；在客户端，gRPC实现了一个stub（可以简单理解为一个client），其提供跟服务端相同的方法。

![img](https:////upload-images.jianshu.io/upload_images/13587608-437291e95f1a4eef.png?imageMogr2/auto-orient/strip|imageView2/2/w/510/format/webp)



**gRPC使用protocol buffers作为接口描述语言（IDL）以及底层的信息交换格式**，一般情况下推荐使用 proto3因为其能够支持更多的语言，并减少一些兼容性的问题。

# 特性

**基于HTTP/2**
 HTTP/2 提供了连接多路复用、双向流、服务器推送、请求优先级、首部压缩等机制。可以节省带宽、降低TCP链接次数、节省CPU，帮助移动设备延长电池寿命等。gRPC 的协议设计上使用了HTTP2 现有的语义，请求和响应的数据使用HTTP Body 发送，其他的控制信息则用Header 表示。
 **IDL使用ProtoBuf**
 gRPC使用ProtoBuf来定义服务，ProtoBuf是由Google开发的一种数据序列化协议（类似于XML、JSON、hessian）。**ProtoBuf能够将数据进行序列化**，并广泛应用在数据存储、通信协议等方面。压缩和传输效率高，语法简单，表达力强。
 **多语言支持（C, C++, Python, PHP, Nodejs, C#, Objective-C、Golang、Java）**
 gRPC支持多种语言，并能够基于语言自动生成客户端和服务端功能库。目前已提供了C版本grpc、Java版本grpc-java 和 Go版本grpc-go，其它语言的版本正在积极开发中，其中，grpc支持C、C++、Node.js、Python、Ruby、Objective-C、PHP和C#等语言，grpc-java已经支持Android开发。
 gRPC已经应用在Google的云服务和对外提供的API中，其主要应用场景如下：

- 低延迟、高扩展性、分布式的系统
- 同云服务器进行通信的移动应用客户端
- 设计语言独立、高效、精确的新协议
- 便于各方面扩展的分层设计，如认证、负载均衡、日志记录、监控等

# grpc优缺点：

**优点：**

- 1、protobuf二进制消息，性能好/效率高（空间和时间效率都很不错）
- 2、proto文件生成目标代码，简单易用
- 3、序列化反序列化直接对应程序中的数据类，不需要解析后在进行映射(XML,JSON都是这种方式)
- 4、支持向前兼容（新加字段采用默认值）和向后兼容（忽略新加字段），简化升级
- 5、支持多种语言（可以把proto文件看做IDL文件）
- 6、Netty等一些框架集成

**缺点：**

- 1、GRPC尚未提供连接池，需要自行实现
- 2、尚未提供“服务发现”、“负载均衡”机制
- 3、因为基于HTTP2，绝大部多数HTTP Server、Nginx都尚不支持，即Nginx不能将GRPC请求作为HTTP请求来负载均衡，而是作为普通的TCP请求。（nginx1.9版本已支持）
- 4、Protobuf二进制可读性差（貌似提供了Text_Fromat功能）
- 5、默认不具备动态特性（可以通过动态定义生成消息类型或者动态编译支持）

# gRPC有四种通信方式:

- 1、 Simple RPC
   简单rpc
   这就是一般的rpc调用，一个请求对象对应一个返回对象
   proto语法：



```cpp
rpc simpleHello(Person) returns (Result) {}
```

- 2、 Server-side streaming RPC
   服务端流式rpc
   一个请求对象，服务端可以传回多个结果对象
   proto语法



```cpp
rpc serverStreamHello(Person) returns (stream Result) {}
```

- 3、 Client-side streaming RPC
   客户端流式rpc
   客户端传入多个请求对象，服务端返回一个响应结果
   proto语法



```cpp
rpc clientStreamHello(stream Person) returns (Result) {}
```

- 4、 Bidirectional streaming RPC
   双向流式rpc
   结合客户端流式rpc和服务端流式rpc，可以传入多个对象，返回多个响应对象
   proto语法



```cpp
rpc biStreamHello(stream Person) returns (stream Result) {}
```

# 服务定义及ProtoBuf

gRPC使用ProtoBuf定义服务， 我们可以一次性的在一个 .proto 文件中定义服务并使用任何支持它的语言去实现客户端和服务器，反过来，它们可以在各种环境中，从云服务器到你自己的平板电脑—— gRPC 帮你解决了不同语言及环境间通信的复杂性。使用 protocol buffers 还能获得其他好处，包括高效的序列号，简单的 IDL 以及容易进行接口更新。

# protoc编译工具

protoc工具可在https://github.com/google/protobuf/releases 下载到源码。 且将protoc的bin目录配置到环境变量中，如下图：

![img](https:////upload-images.jianshu.io/upload_images/13587608-5b86587acc996229.png?imageMogr2/auto-orient/strip|imageView2/2/w/399/format/webp)



实际开发中一般都通过在idea上配置`com.google.protobuf`插件进行开发，这一点在https://github.com/grpc/grpc-java上的文档有详细说明，如果使用gradle进行项目构建的话，https://github.com/google/protobuf-gradle-plugin上有protobuf-gradle-plugin插件的详细使用说明。

# protobuf语法

- 1、syntax = “proto3”;
   文件的第一行指定了你使用的是proto3的语法：如果你不指定，protocol buffer 编译器就会认为你使用的是proto2的语法。这个语句必须出现在.proto文件的非空非注释的第一行。

- 2、message SearchRequest {……}
   message 定义实体，c/c++/go中的结构体，php中类

- 3、基本数据类型

  ![img](https:////upload-images.jianshu.io/upload_images/13587608-fabc080282b4fd72.png?imageMogr2/auto-orient/strip|imageView2/2/w/706/format/webp)

  image.png

- 4、注释符号： 双斜线，如：//xxxxxxxxxxxxxxxxxxx

- 5、字段唯一数字标识（用于在二进制格式中识别各个字段，上线后不宜再变动）：Tags
   1到15使用一个字节来编码，包括标识数字和字段类型（你可以在Protocol Buffer 编码中查看更多详细）；16到2047占用两个字节。因此定义proto文件时应该保留1到15，用作出现最频繁的消息类型的标识。记得为将来会继续增加并可能频繁出现的元素留一点儿标识区间，也就是说，不要一下子把1—15全部用完，为将来留一点儿。
   标识数字的合法范围：最小是1，最大是 229 - 1，或者536,870,911。
   另外，不能使用19000 到 19999之间的数字(FieldDescriptor::kFirstReservedNumber through FieldDescriptor::kLastReservedNumber)，因为它们被Protocol Buffers保留使用

- 6、字段修饰符：
   required：值不可为空
   optional：可选字段
   singular：符合语法规则的消息包含零个或者一个这样的字段（最多一个）
   repeated：一个字段在合法的消息中可以重复出现一定次数（包括零次）。重复出现的值的次序将被保留。在proto3中，重复出现的值类型字段默认采用压缩编码。你可以在这里找到更多关于压缩编码的东西： Protocol Buffer Encoding。
   默认值： optional PhoneType type = 2 [default = HOME];
   proto3中，省略required,optional,singular，由protoc自动选择。

- 7、代理类生成
   1)、C++, 每一个.proto 文件可以生成一个 .h 文件和一个 .cc 文件
   2)、Java, 每一个.proto文件可以生成一个 .java 文件
   3)、Python, 每一个.proto文件生成一个模块，其中为每一个消息类型生成一个静态的描述器，在运行时，和一个metaclass一起使用来创建必要的Python数据访问类
   4)、Go, 每一个.proto生成一个 .pb.go 文件
   5)、Ruby, 每一个.proto生成一个 .rb 文件
   6)、Objective-C, 每一个.proto 文件可以生成一个 pbobjc.h 和一个pbobjc.m 文件
   7)、C#, 每一个.proto文件可以生成一个.cs文件.
   8)、php, 每一个message消息体生成一个.php类文件，并在GPBMetadata目录生成一个对应包名的.php类文件，用于保存.proto的二进制元数据。

- 8、字段默认值

- strings, 默认值是空字符串（empty string）
- bytes, 默认值是空bytes（empty bytes）
- bools, 默认值是false
- numeric, 默认值是0
- enums, 默认值是第一个枚举值（value必须为0）
- message fields, the field is not set. Its exact value is langauge-dependent. See the generated code guide for details.
- repeated fields，默认值为empty，通常是一个空list

- 9、枚举



```rust
// 枚举类型，必须从0开始，序号可跨越。同一包下不能重名，所以加前缀来区别
enum WshExportInstStatus {
    INST_INITED = 0;
    INST_RUNNING = 1;
    INST_FINISH = 2;
    INST_FAILED = 3;
}
```

- 10、Maps字段类型



```cpp
map<key_type, value_type> map_field = N;
```

其中key_type可以是任意Integer或者string类型（所以，除了floating和bytes的任意标量类型都是可以的）value_type可以是任意类型。
 例如，如果你希望创建一个project的映射，每个Projecct使用一个string作为key，你可以像下面这样定义：



```cpp
map<string, Project> projects = 3;
```

Map的字段可以是repeated。
 序列化后的顺序和map迭代器的顺序是不确定的，所以你不要期望以固定顺序处理Map
 当为.proto文件产生生成文本格式的时候，map会按照key 的顺序排序，数值化的key会按照数值排序。
 从序列化中解析或者融合时，如果有重复的key则后一个key不会被使用，当从文本格式中解析map时，如果存在重复的key。

- 11、默认值
   字符串类型默认为空字符串
   字节类型默认为空字节
   布尔类型默认false
   数值类型默认为0值
   enums类型默认为第一个定义的枚举值，必须是0
- 12、服务
   服务使用service{}包起来，每个方法使用rpc起一行申明，一个方法包含一个请求消息体和一个返回消息体



```cpp
service HelloService {
  rpc SayHello (HelloRequest) returns (HelloResponse);
}
message HelloRequest {
  string greeting = 1;
}

message HelloResponse {
  string reply = 1;
}
```

[更多protobuf参考(google)](https://developers.google.com/protocol-buffers/docs/reference/overview?hl=zh-cn)
 [更多protobuf参考(csdn)](http://blog.csdn.net/u011518120/article/details/54604615)

# 对于开发者而言：

- 1、需要使用protobuf定义接口，即.proto文件
- 2、然后使用compile工具生成特定语言的执行代码，比如JAVA、C/C++、Python等。类似于thrift，为了解决跨语言问题。
- 3、启动一个Server端，server端通过侦听指定的port，来等待Client链接请求，通常使用Netty来构建，GRPC内置了Netty的支持。
- 4、启动一个或者多个Client端，Client也是基于Netty，Client通过与Server建立TCP长链接，并发送请求；Request与Response均被封装成HTTP2的stream Frame，通过Netty Channel进行交互。

# 实例

## 引入maven或gradle的jar依赖



```jsx
//maven
<dependency>
  <groupId>io.grpc</groupId>
  <artifactId>grpc-netty-shaded</artifactId>
  <version>1.19.0</version>
</dependency>
<dependency>
  <groupId>io.grpc</groupId>
  <artifactId>grpc-protobuf</artifactId>
  <version>1.19.0</version>
</dependency>
<dependency>
  <groupId>io.grpc</groupId>
  <artifactId>grpc-stub</artifactId>
  <version>1.19.0</version>
</dependency>

//gradle
compile 'io.grpc:grpc-netty-shaded:1.19.0'
compile 'io.grpc:grpc-protobuf:1.19.0'
compile 'io.grpc:grpc-stub:1.19.0'
```

## 生成的代码步骤

- 1、对于基于protobuf的codegen，将原型文件Xxx.proto放在src/main/proto 和src/test/proto目录中。
- 2、protobuf的插件的使用
   对于使用Maven构建系统集成基于protobuf的，代码生成，您可以使用[protobuf-maven-plugin](https://www.xolstice.org/protobuf-maven-plugin/)



```xml
<build>
  <extensions>
    <extension>
      <groupId>kr.motd.maven</groupId>
      <artifactId>os-maven-plugin</artifactId>
      <version>1.5.0.Final</version>
    </extension>
  </extensions>
  <plugins>
    <plugin>
      <groupId>org.xolstice.maven.plugins</groupId>
      <artifactId>protobuf-maven-plugin</artifactId>
      <version>0.5.1</version>
      <configuration>
        <protocArtifact>com.google.protobuf:protoc:3.6.1:exe:${os.detected.classifier}</protocArtifact>
        <pluginId>grpc-java</pluginId>
        <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.19.0:exe:${os.detected.classifier}</pluginArtifact>
      </configuration>
      <executions>
        <execution>
          <goals>
            <goal>compile</goal>
            <goal>compile-custom</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

对于与Gradle构建系统集成的基于[protobuf的codegen](https://github.com/google/protobuf-gradle-plugin)，您可以在build.gradle文件中增加[protobuf-gradle-plugin](https://github.com/google/protobuf-gradle-plugin)插件：



```csharp
apply plugin: 'com.google.protobuf'

buildscript {
  repositories {
    mavenCentral()
  }
  dependencies {
    classpath 'com.google.protobuf:protobuf-gradle-plugin:0.8.5'
  }
}

protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:3.6.1"
    }
    plugins {
        grpc {
            artifact = 'io.grpc:protoc-gen-grpc-java:1.19.0'
        }
    }

    //生成的grpc的message类的路径
    generatedFilesBaseDir = "src"
    
    generateProtoTasks {
        all()*.plugins {
            grpc {
                //生成的grpc的service服务类的路径
                outputSubDir = 'java'
            }
        }
    }
}
```

然后在idea的控制台上输入：`gradle generateProto`即可生成grpc代码

![img](https:////upload-images.jianshu.io/upload_images/13587608-cfa9fa7d86098470.png?imageMogr2/auto-orient/strip|imageView2/2/w/421/format/webp)

image.png



# 1、Simple RPC  简单rpc，这就是一般的rpc调用，一个请求对象对应一个返回对象

**定义Student.proto文件**



```cpp
syntax = "proto3";

package com.yibo.proto;

option java_package = "com.yibo.proto";
option java_outer_classname = "StudentProto";
option java_multiple_files = true;

service StudentService {
    rpc GetRealnameByUsername (MyRequest) returns (MyResponse) {
    }
}

message MyRequest {
    string username = 1;
}

message MyResponse {
    string realname = 1;
}
```

**服务端实现proto中的接口：**



```java
public class StudentServiceImpl extends StudentServiceGrpc.StudentServiceImplBase {

    @Override
    public void getRealnameByUsername(MyRequest request, StreamObserver<MyResponse> responseObserver) {
        System.out.println("接收到客户端信息：" + request.getUsername());

        //将返回结果构造完，并且返回给客户端
        responseObserver.onNext(MyResponse.newBuilder().setRealname("张三").build());
        //告诉客户端方法执行完毕
        responseObserver.onCompleted();
    }
}
```

**grpc服务端的实现**



```java
public class GrpcServer {

    private Server server;

    private void start() throws IOException {
        this.server = ServerBuilder.forPort(8899).addService(new StudentServiceImpl()).build().start();

        System.out.println("server started!");

        //这是在服务端jvm关闭之前主动退出grpc服务，且关闭其相应的资源
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("关闭jvm");
            GrpcServer.this.stop();
        }));
    }

    private void stop(){
        if(null != this.server){
            this.server.shutdown();
        }
    }

    //让服务启动后处于等待状态，不然服务已启动马上就停止了
    private void awaitTermination() throws InterruptedException {
        if(null != this.server){
            this.server.awaitTermination();
        }
    }

    public static void main(String[] args) throws InterruptedException, IOException {
        GrpcServer grpcServer = new GrpcServer();

        grpcServer.start();
        grpcServer.awaitTermination();
    }
}
```

**grpc客户端的实现**



```csharp
public class GrpcClient {

    public static void main(String[] args) {
        ManagedChannel managedChannel = ManagedChannelBuilder.forAddress("localhost",8899)
                .usePlaintext().build();
        StudentServiceGrpc.StudentServiceBlockingStub blockingStub = StudentServiceGrpc.newBlockingStub(managedChannel);

        MyResponse myResponse = blockingStub.getRealnameByUsername(MyRequest.newBuilder().setUsername("李四").build());
        System.out.println(myResponse.getRealname());
    }
}
```

# 2、Server-side streaming RPC 服务端流式rpc，一个请求对象，服务端可以传回多个结果对象

**定义Student.proto文件**



```go
syntax = "proto3";

package com.yibo.proto;

option java_package = "com.yibo.proto";
option java_outer_classname = "StudentProto";
option java_multiple_files = true;

service StudentService {
    rpc GetStudentByAge (StudentRequest) returns (stream StudentResponse){}
}

message StudentRequest {
    int32 age = 1;
}

message StudentResponse {
    string name = 1;
    int32 age = 2;
    string city = 3;
}
```

**服务端实现proto中的接口：**



```java
public class StudentServiceImpl extends StudentServiceGrpc.StudentServiceImplBase {
    @Override
    public void getStudentByAge(StudentRequest request, StreamObserver<StudentResponse> responseObserver) {
        System.out.println("接收到客户端信息：" + request.getAge());

        responseObserver.onNext(StudentResponse.newBuilder().setName("张三").setAge(50).setCity("北京").build());
        responseObserver.onNext(StudentResponse.newBuilder().setName("李四").setAge(40).setCity("上海").build());
        responseObserver.onNext(StudentResponse.newBuilder().setName("王五").setAge(30).setCity("深圳").build());
        responseObserver.onNext(StudentResponse.newBuilder().setName("赵六").setAge(20).setCity("重庆").build());
        responseObserver.onCompleted();
    }
}
```

**grpc服务端的实现沿用Simple RPC例子里面的服务端实现**

**grpc客户端的实现**



```csharp
public class GrpcClient {

    public static void main(String[] args) {
        ManagedChannel managedChannel = ManagedChannelBuilder.forAddress("localhost",8899)
                .usePlaintext().build();
        StudentServiceGrpc.StudentServiceBlockingStub blockingStub = StudentServiceGrpc.newBlockingStub(managedChannel);
        Iterator<StudentResponse> iterator = blockingStub.getStudentByAge(StudentRequest.newBuilder().setAge(20).build());
        while(iterator.hasNext()){
            StudentResponse studentResponse = iterator.next();
            System.out.println(studentResponse.getName() + "：" + studentResponse.getName() + "：" + studentResponse.getCity());
        }
    }
}
```

# 3、Client-side streaming RPC 客户端流式rpc，客户端传入多个请求对象，服务端返回一个响应结果

**定义Student.proto文件**



```go
syntax = "proto3";

package com.yibo.proto;

option java_package = "com.yibo.proto";
option java_outer_classname = "StudentProto";
option java_multiple_files = true;

service StudentService {
    rpc GetStudentsWrapperByAges (stream StudentRequest) returns (StudentResponseList){}
}

message StudentRequest {
    int32 age = 1;
}

message StudentResponse {
    string name = 1;
    int32 age = 2;
    string city = 3;
}

message StudentResponseList {
    repeated StudentResponse studentResponse = 1;
}
```

**服务端实现proto中的接口：**



```java
public class StudentServiceImpl extends StudentServiceGrpc.StudentServiceImplBase {
    @Override
    public StreamObserver<StudentRequest> getStudentsWrapperByAges(StreamObserver<StudentResponseList> responseObserver) {
        return new StreamObserver<StudentRequest>() {
            @Override
            public void onNext(StudentRequest value) {
                System.out.println("onNext" + value.getAge());
            }

            @Override
            public void onError(Throwable t) {
                System.out.println(t.getMessage());
            }

            @Override
            public void onCompleted() {
                StudentResponse studentResponse1 = StudentResponse.newBuilder().setName("张三").setAge(20).setCity("深圳").build();
                StudentResponse studentResponse2 = StudentResponse.newBuilder().setName("李四").setAge(22).setCity("重庆").build();
                StudentResponseList studentResponseList = StudentResponseList.newBuilder()
                        .addStudentResponse(studentResponse1).addStudentResponse(studentResponse2).build();
                responseObserver.onNext(studentResponseList);
                responseObserver.onCompleted();
            }
        };
    }
}
```

**grpc服务端的实现沿用Simple RPC例子里面的服务端实现**

**grpc客户端的实现**



```csharp
public class GrpcClient {

    public static void main(String[] args) {
        ManagedChannel managedChannel = ManagedChannelBuilder.forAddress("localhost",8899)
                .usePlaintext().build();
        StudentServiceGrpc.StudentServiceStub stub = StudentServiceGrpc.newStub(managedChannel);    

        StreamObserver<StudentResponseList> studentResponseListStreamObserver = new StreamObserver<StudentResponseList>() {
            @Override
            public void onNext(StudentResponseList value) {
                value.getStudentResponseList().forEach(studentResponse -> {
                    System.out.println(studentResponse.getName() + "：" + studentResponse.getAge() + "：" + studentResponse.getCity());
                });
            }

            @Override
            public void onError(Throwable t) {
                System.out.println(t.getMessage());
            }

            @Override
            public void onCompleted() {
                System.out.println("completed!");
            }
        };

        StreamObserver<StudentRequest> studentRequestStreamObserver = stub.getStudentsWrapperByAges(studentResponseListStreamObserver);
        studentRequestStreamObserver.onNext(StudentRequest.newBuilder().setAge(10).build());
        studentRequestStreamObserver.onNext(StudentRequest.newBuilder().setAge(20).build());
        studentRequestStreamObserver.onNext(StudentRequest.newBuilder().setAge(30).build());
        studentRequestStreamObserver.onNext(StudentRequest.newBuilder().setAge(40).build());
        studentRequestStreamObserver.onCompleted();

        try {
            //因为grpc调用参数为stream的时候，都是采取异步处理，客户端程序不会等待结果响应
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

# 4、 Bidirectional streaming RPC 双向流式rpc，结合客户端流式rpc和服务端流式rpc，可以传入多个对象，返回多个响应对象

**定义Student.proto文件**



```cpp
syntax = "proto3";

package com.yibo.proto;

option java_package = "com.yibo.proto";
option java_outer_classname = "StudentProto";
option java_multiple_files = true;

service StudentService {
    rpc BiTalk (stream StreamRequest) returns (stream StreamResponse){}
}

message StreamRequest {
    string request_info = 1;
}

message StreamResponse {
    string response_info = 1;
}
```

**服务端实现proto中的接口：**



```java
public class StudentServiceImpl extends StudentServiceGrpc.StudentServiceImplBase {

    @Override
    public StreamObserver<StreamRequest> biTalk(StreamObserver<StreamResponse> responseObserver) {
        return new StreamObserver<StreamRequest>() {
            @Override
            public void onNext(StreamRequest value) {
                System.out.println("onNext" + value.getRequestInfo());
                responseObserver.onNext(StreamResponse.newBuilder().setResponseInfo(UUID.randomUUID().toString()).build());
            }

            @Override
            public void onError(Throwable t) {
                System.out.println(t.getMessage());
            }

            @Override
            public void onCompleted() {
                responseObserver.onCompleted();
            }
        };
    }
}
```

**grpc服务端的实现沿用Simple RPC例子里面的服务端实现**

**grpc客户端的实现**



```csharp
public class GrpcClient {

    public static void main(String[] args) {
        ManagedChannel managedChannel = ManagedChannelBuilder.forAddress("localhost",8899)
                .usePlaintext().build();

        StudentServiceGrpc.StudentServiceStub stub = StudentServiceGrpc.newStub(managedChannel);

        StreamObserver<StreamRequest> streamRequestStreamObserver = stub.biTalk(new StreamObserver<StreamResponse>() {
            @Override
            public void onNext(StreamResponse value) {
                System.out.println(value.getResponseInfo());
            }

            @Override
            public void onError(Throwable t) {
                System.out.println(t.getMessage());
            }

            @Override
            public void onCompleted() {
                System.out.println("completed!");
            }
        });

        for(int i = 0 ;i < 10 ; i++){
            streamRequestStreamObserver.onNext(StreamRequest.newBuilder().setRequestInfo(LocalDateTime.now().toString()).build());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        try {
            //因为grpc调用参数为stream的时候，都是采取异步处理，客户端程序不会等待结果响应
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

至此，grpc中grpc-java的大体内容与用法就基本完结了，grpc四种通信方式中，第一种Simple RPC方式是用的最多的，至于其他三种通信方式什么时候用，也是根据具体业务场景而言的。

# grpc在grpc-node的客户端和服务端的实例。

## 在WebStorm上创建一个项目，并创建package.json文件

![img](https:////upload-images.jianshu.io/upload_images/13587608-d17571771abd2596.png?imageMogr2/auto-orient/strip|imageView2/2/w/572/format/webp)

image.png

然后通过`npm install`将依赖下载到本地

![img](https:////upload-images.jianshu.io/upload_images/13587608-597748742ca10486.png?imageMogr2/auto-orient/strip|imageView2/2/w/1085/format/webp)

image.png



相比较于Java等语言来说，node.js在grpc的实现过程中有一些明显的不同，具体体现代码的编写方式上，那么grpc在node.js的实现中有2种情况，即动态代码生成和静态代码生成。

- 动态代码生成：不需要提前由proto文件生成对应的js代码，通过提前指定proto文件的位置，在运行的过程中动态的取生成proto文件对应的文件。动态生成包括客户端的动态生成和服务器端的动态生成。
- 静态代码生成：和前面的Java类似，通过grpc针对node.js的编译器，提前由proto文件取生成对应的js文件。

## 定义Student.proto文件



```cpp
syntax = "proto3";

package com.yibo.proto;

option java_package = "com.yibo.proto";
option java_outer_classname = "StudentProto";
option java_multiple_files = true;

service StudentService {
    rpc GetRealnameByUsername (MyRequest) returns (MyResponse) {}

    rpc GetStudentByAge (StudentRequest) returns (stream StudentResponse){}

    rpc GetStudentsWrapperByAges (stream StudentRequest) returns (StudentResponseList){}

    rpc BiTalk (stream StreamRequest) returns (stream StreamResponse){}
}

message MyRequest {
    string username = 1;
}

message MyResponse {
    string realname = 1;
}

message StudentRequest {
    int32 age = 1;
}

message StudentResponse {
    string name = 1;
    int32 age = 2;
    string city = 3;
}

message StudentResponseList {
    repeated StudentResponse studentResponse = 1;
}

message StreamRequest {
    string request_info = 1;
}

message StreamResponse {
    string response_info = 1;
}
```

## 动态代码生成

**创建grpc-node的客户端**
 在根目录下创建app目录，然后创建grpcClient.js



```jsx
var PROTO_FILE_PATH = '/Users/yibo/front_end/rpc_demo/proto/Student.proto';
var grpc = require('grpc');
var grpcService = grpc.load(PROTO_FILE_PATH).com.yibo.proto;

var client = new grpcService.StudentService('localhost:8899',grpc.credentials.createInsecure());

client.getRealnameByUsername({username : 'lisi'},function(error,respData){
    console.log(respData);
});
```

**创建grpc-node的服务端**
 在app目录下，创建grpcServer.js



```jsx
var PROTO_FILE_PATH = '/Users/yibo/front_end/rpc_demo/proto/Student.proto';
var grpc = require('grpc');
var grpcService = grpc.load(PROTO_FILE_PATH).com.yibo.proto;

var server = new grpc.Server();

server.addService(grpcService.StudentService.service,{
    getRealnameByUsername: getRealnameByUsername,
    getStudentByAge: getStudentByAge,
    getStudentsWrapperByAges: getStudentsWrapperByAges,
    biTalk: biTalk
});

server.bind('localhost:8899',grpc.ServerCredentials.createInsecure());
server.start();

function getRealnameByUsername(call,callback){
    console.log("username：" + call.request.username);
    callback(null, {realname : '张三'});
}

function getStudentByAge,(call,callback){
    //演示例子用不上，所有采用空实现
}

function getStudentsWrapperByAges:(call,callback){
    //演示例子用不上，所有采用空实现
}

function biTalk(call,callback){
    //演示例子用不上，所有采用空实现
}
```

## 静态代码生成（和grpc-java类似）

首先要保证本机装有protoc
 grpc_node_plugin插件的安装说明：https://github.com/grpc/grpc/tree/v1.19.0/examples/node/static_codegen
 **在WebStorm控制台上敲入`npm install -g grpc-tools`安装grpc_node_plugin插件**

![img](https:////upload-images.jianshu.io/upload_images/13587608-0aa5cb69fdbee921.png?imageMogr2/auto-orient/strip|imageView2/2/w/855/format/webp)

image.png



然后在在WebStorm控制台上执行(可能会报`static_codegen/: No such file or directory`错误，那么需要在根目录下创建static_codegen目录)：



```ruby
grpc_tools_node_protoc --js_out=import_style=commonjs,binary:static_codegen/ --grpc_out=static_codegen --plugin=protoc-gen-grpc=/usr/local/bin/grpc_tools_protoc_plugin node proto/Student.proto
```

执行命令后，如果没有任何提示表示成功，如下图：



![img](https:////upload-images.jianshu.io/upload_images/13587608-5f9c5b0294585008.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

image.png

**创建grpc-node的客户端**
 在app目录下，创建grpcClient2.js



```jsx
var service = require('../static_codegen/proto/Student_grpc_pb');
var messages = require('../static_codegen/proto/Student_pb');

var grpc = require('grpc');

var client = new service.StudentServiceVlient('localhost:8899',grpc.credentials.createInsecure());

var request = new message.MyRequest();
request.setUsername('lisi');

client.getRealNameByUsername(request,function(error,respData){
    console.log(respData.getRealname());
});
```

**创建grpc-node的服务端**
 在app目录下，创建grpcServer2.js



```jsx
var service = require('../static_codegen/proto/Student_grpc_pb');
var messages = require('../static_codegen/proto/Student_pb');

var grpc = require('grpc');

var server = new grpc.Server();

server.addService(service.Student.ServiceService,{
    getRealnameByUsername: getRealnameByUsername,
    getStudentByAge: getStudentByAge,
    getStudentsWrapperByAges: getStudentsWrapperByAges,
    biTalk: biTalk
});

server.bind('localhost:8899',grpc.ServerCredentials.createInsecure());
server.start();

function getRealnameByUsername(call,callback){
    console.log('request：' + call.request.getUsername());

    var myResponse = new message.MyResponse();
    myResponse.setRealname('王五');
    callback(null,myResponse);
}

function getStudentByAge,(call,callback){
    //演示例子用不上，所有采用空实现
}

function getStudentsWrapperByAges:(call,callback){
    //演示例子用不上，所有采用空实现
}

function biTalk(call,callback){
    //演示例子用不上，所有采用空实现
}
```

综合上面的grpc-java和grpc-node的例子，grpc就可以实现跨语言异构平台之间的rpc调用。而且grpc是一个很有前途的跨语言异构平台rpc框架，SpringBoot已经提供了整合，加上google的背书，grpc未来一定会得到更广泛的应用。

# 参考：

## [grpc](https://blog.csdn.net/xuduorui/article/details/78278808)

## [GRPC快速入门](https://chenjiehua.me/python/grpc-quick-start.html)

## [gRPC](https://www.cnblogs.com/dadadechengzi/p/8819821.html)

## [浅析gRPC](https://www.colabug.com/4616436.html)



作者：小波同学
链接：https://www.jianshu.com/p/7392406e2450
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。