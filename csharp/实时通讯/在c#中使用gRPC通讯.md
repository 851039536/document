参考 : [C#封装GRPC类库及调用简单实例 - wtc87 - 博客园 (cnblogs.com)](https://www.cnblogs.com/wtc87/p/17002800.html)

包括：GRPC文件的创建生成、服务端和客户端函数类库的封装、创建服务端和客户端调用测试。

### **创建并生成GRPC服务文件**

1.  创建新项目控制台应用 , 项目名称(MgRPC)
2.  安装三个nuget包 Google.Protobuf , Grpc.Core , Grpc.Tools
3.  项目添加新建项，选择类，修改名称为Link.proto，添加后把Link.proto里面内容清空

#### 定义Protocol

添加代码。测试实例为服务端和客户端传输字符串消息，只定义了一个方法（客户端调用，服务端重写），传输内容包括请求字符串和回复字符串。此处可自行定义。

```csharp

syntax = "proto3";//proto3 是 Protocol Buffers 的第三个版本

// 指定了生成的 C# 代码的命名空间为 LinkService。当使用 protobuf 编译器 (protoc) 将这个 .proto 文件转换为 C# 代码时，生成的类将位于 LinkService 命名空间中
option csharp_namespace = "LinkService";

//定义了一个名为 Link 的 gRPC 服务。在 gRPC 中，服务是由一个或多个 RPC 方法组成的
service Link
{
	//定义了一个 RPC 方法。这个方法名为 GetMessage，它接受一个 Mes 类型的消息作为参数，
	//并返回一个 Mes 类型的消息。在 gRPC 中，客户端可以调用这个方法，并发送一个 Mes 消息给服务端，然后服务端会处理这个消息并返回一个 Mes 消息给客户端。
	rpc GetMessage(Mes) returns (Mes);
}

//定义了一个名为 Mes 的消息类型。在 protobuf 中，消息是由一系列字段组成的，每个字段都有一个名称、一个类型和一个标识符。
message Mes
{
    // 客户端发送
	// 定义了一个名为 StrRequest 的字段，类型为 string，标识符为 1。这个标识符在消息内部是唯一的，并且一旦分配就不能更改，因为它被用于序列化和反序列化过程中的字段识别。
	string StrRequest = 1;
	// 同样定义了一个名为 StrReply 的字段，类型为 string，标识符为 2。(服务端回复)
	string StrReply = 2;
}
```

#### 设置Link.proto

右键Link.proto文件选择属性，生成操作选择如图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f63ea76206f4bc09995da665841075e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=651\&h=193\&s=14038\&e=png\&b=f6f5f5)

#### 生成cs文件代码

生成解决方案。在下图路径得到自动生成的两个类。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eeb6de323a454517a0f000e78540993b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=836\&h=260\&s=29900\&e=png\&b=ffffff)

至此，获得GRPC服务需要的三个文件：Link.proto、Link.cs、LinkGrpc.cs。可以将这三个文件放在一个项目中直接使用，需要重写一下服务端方法、创建服务端和客户端的启动方法。但是如果不同的项目软件之间通讯需要各自如此开发。可以先封装成一个GRPC类库供其他项目直接调用。

### **服务端和客户端类库的封装**

1.  创建类库(.NET Framework)项目 , 项目名称(GrpcLink)
2.  项目添加现有项，上面获得的三个文件(Link.cs , LinkGrpc.cs)。安装nuget包：Grpc.Core和Google.Protobuf。
3.  创建两个类：LinkFunc用于放此类库可用于外部引用调用的方法。LinkServerFunc基于Link.LinkBase，用于重写在proto文件中定义的方法。

对于不同的项目，在客户端请求时，服务端要根据自身情况回复想回的内容，因此可以提供一个委托供外部自行开发回复函数。

在LinkFunc类中定义如下：

```csharp
public static Func<string, string> ReplyMes;
```

#### LinkServerFunc

在LinkServerFunc类重写GetMessage方法如下

```csharp
using Grpc.Core;
using LinkService;
using System.Threading.Tasks;
using static LinkService.Link;

namespace GrpcLink
{
    /// <summary>
    /// 重写在proto文件中定义的方法
    /// </summary>
    public class LinkServerFunc : LinkBase
    {
        public override Task<Mes> GetMessage(Mes request,ServerCallContext context)
        {
            Mes mes = new Mes();
            mes.StrReply = LinkFunc.ReplyMes(request.StrRequest);
            return Task.FromResult(mes);
        }
    }
}
```

#### LinkFunc

```csharp
using Grpc.Core;
using LinkService;
using System;
using static LinkService.Link;

namespace GrpcLink
{
    public class LinkFunc
    {
        /// <summary>
        /// 用于服务端回复委托
        /// </summary>
        public static Func<string,string> ReplyMes;

        // 定义服务端和客户端
        public static Server LinkServer;
        public static LinkClient LinkClient;

        /// <summary>
        ///  服务端启动
        /// </summary>
        /// <param name="host"></param>
        /// <param name="port"></param>
        public static void LinkServerStart(string host,int port)
        {
            LinkServer = new Server
            {
                Services =
                    {
                        BindService(new LinkServerFunc())
                    },
                Ports = { new ServerPort(host,port,ServerCredentials.Insecure) }
            };
            LinkServer.Start();
        }

        /// <summary>
        ///  服务端关闭
        /// </summary>
        public static void LinkServerClose()
        {
            LinkServer?.ShutdownAsync().Wait();
        }

        /// <summary>
        /// 客户端启动
        /// </summary>
        /// <param name="strIp"></param>
        public static void LinkClientStart(string strIp)
        {
            Channel prechannel = new Channel(strIp,ChannelCredentials.Insecure);
            LinkClient = new LinkClient(prechannel);
        }

        /// <summary>
        /// 客户端发送消息函数
        /// </summary>
        /// <param name="strRequest"></param>
        /// <returns></returns>
        public static string SendMes(string strRequest)
        {
            Mes mes = new Mes();
            mes.StrRequest = strRequest;
            var res = LinkClient.GetMessage(mes);
            return res.StrReply;
        }
    }
}
```

#### 生成引用库

生成解决方案。Debug中可以得到项目的dll文件GrpcLink.dll，其他项目可以引用使用了。

### **创建服务端和客户端调用测试**

1.  创建两个控制台(.NET Framework)项目 , 项目名称TestCilent , TestServer
2.  将上述GrpcLink.dll文件分别放入两个项目中，并添加dll引用。 如果没有则安装nuget包：Grpc.Core和Google.Protobuf(此次测试没有安装)

#### TestServer服务端

```csharp
using GrpcLink;
using System;
using System.Threading;

namespace TestServer
{
    /// <summary>
    /// 测试服务端
    /// </summary>
    internal class Program
    {
        static void Main(string[] args)
        {
            LinkFunc.LinkServerStart("127.0.0.1",9008);
            Thread.Sleep(500);
            LinkFunc.ReplyMes = ReplyMes;
            Console.ReadKey();
        }


        /// <summary>
        /// 接收到客户端信息后回复
        /// </summary>
        /// <param name="strRequest">客户端发送过来的内容</param>
        /// <returns></returns>
        public static string ReplyMes(string strRequest)
        {
            Console.WriteLine("接收到:" + strRequest);
            switch (strRequest)
            {
                case "1":
                return "Server识别到1";
                case "2":
                return "Server识别到2";
                case "测试":
                return "开始测试"; 
                case "连接服务端":
                return "true";
            }
            return "Server未识别到指定参数";
        }
    }
}

```

#### TestCilent客户端

```csharp
using GrpcLink;
using System;
using System.Threading;

namespace TestCilent
{
    internal class Program
    {
        static void Main(string[] args)
        {
            Thread.Sleep(2000);
            LinkFunc.LinkClientStart("127.0.0.1:9008");
            Console.WriteLine("连接服务端中");
            string conn = LinkFunc.SendMes("连接服务端");

            if (conn.Equals("true"))
            {
                Console.WriteLine("连接服务端成功!");


                for (int i = 0 ; i < 10 ; i++)
                {
                    Thread.Sleep(1000);
                    Console.WriteLine("输入测试内容..");
                    var line = Console.ReadLine();
                    var ret = LinkFunc.SendMes(line);
                    //获取到服务端返回的值
                    Console.WriteLine(ret);

                }
            }

            Console.WriteLine("连接失败 , 即将退出");
            Thread.Sleep(2000);
        }
    }
}

```

### 测试图例

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8cd2b091e00642f8b6e104e819ae56c3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=664\&h=322\&s=17353\&e=png\&b=0c0c0c) 