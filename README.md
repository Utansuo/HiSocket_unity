# HiSocket_unity


尚未完成

Haven't finish


开源地址：[https://github.com/hiramtan/HiSocket_unity](https://github.com/hiramtan/HiSocket_unity)

## To Do List
- [x] ipv6支持
- [x] 主线程连接
- [x] protobuf
- [x] aes加密
- [] 数据压缩
- [x] 多线程连接
- [] 断线重连
- [] 消息缓存队列
- [] 压力测试
- [] 兼容性测试


####
概述:
-------------
第一版已完成，包含如下功能：
- [x] Ipv6支持
- [x] socket连接，收发，断开
- [x] 套接字粘包拆包，消息包的封装，解析
- [x] 消息回调
- [x] 字节消息
- [x] protobuf消息
- [x] aes加密

##
功能说明:
-------------
Tcp socket收发逻辑通用，但是消息包的定义每家各不相同（长度，标识符，时间戳，加密字符..），逻辑设计上也尽量将这部分隔离，方便自定义消息格式。

（如果只需要socket收发逻辑，不需要一整套的消息收发机制，可以只保留工程中的Network文件夹）。

建议采用整套逻辑，套接字的封装解析都不需要做再额外处理。

源码中提供了两种消息结构：字节消息和protobuf消息，可以在unity中定义全局宏定义或者在msgmanager中选择使用哪种方式（默认protobuf）。
源码中提供了两种收发方式：多线程收发和主线程收发，可以在unity中定义全局宏定义或者在clienttcp中选在（默认主线程）。

###
消息定义说明：
-------------

字节消息结构：

[![](https://i1.wp.com/hiramtan.files.wordpress.com/2017/05/11112.png)](https://i1.wp.com/hiramtan.files.wordpress.com/2017/05/11112.png)


Protobuf消息结构：

[![](https://i1.wp.com/hiramtan.files.wordpress.com/2017/05/3332.png?ssl=1&w=450)](https://i1.wp.com/hiramtan.files.wordpress.com/2017/05/3332.png?ssl=1&w=450)

如果项目同时支持字节消息和protobuf消息，可以修改成如下结构：

[![](https://hiramtan.files.wordpress.com/2017/05/3332.png)](https://hiramtan.files.wordpress.com/2017/05/3332.png)

##
示例代码如下：
-------------
``` C#
public class Example : MonoBehaviour
{
    // Use this for initialization
    void Start()
    {
        //registe bytes msg
        MsgManager.Instance.RegisterMsg(110, OnByteMsg);
        //you can registe many msg here
        //....

        //registe protobuf msg
        MsgManager.Instance.RegisterMsg(typeof(TestProtobufStruct).FullName, OnProtobufMsg);
        //....

        //connect(prefer host names)
        ClientTcp socket = new ClientTcp();
        bool tempIsConnect = socket.Connect("www.google.com", 111);
        Debug.Log(tempIsConnect);

        // send byte msg
        MsgByte tempMsg1 = new MsgByte(110);//110 is proto id
        tempMsg1.Write<int>(100);//write msg's body
        tempMsg1.Write("hello");//write msg's body
        tempMsg1.Flush();//send

        //send protobuf msg
        TestProtobufStruct testProtobufStruct = new TestProtobufStruct();
        testProtobufStruct.x = 100;
        testProtobufStruct.y = "hello";
        MsgProtobuf tempMsg2 = new MsgProtobuf();
        tempMsg2.Write(testProtobufStruct);
        tempMsg2.Flush();//send
    }

    void OnByteMsg(MsgBase param)
    {
        var test = param as MsgByte;
        int temp1 = test.Read<int>(); //100
        string temp2 = test.Read<string>(5); //"hello"

        Debug.Log(temp1 + temp2);
    }

    void OnProtobufMsg(MsgBase param)
    {
        var test = param as MsgProtobuf;
        var test2 = test.Read<TestProtobufStruct>();

        int temp1 = test2.x;//100
        string temp2 = test2.y;//"hello"
        Debug.Log(temp1 + temp2);
    }
}
public class TestProtobufStruct
{
    public int x;
    public string y;
}
 ```

###
Ipv6支持说明：
-------------
微软提供了很多接口测试当前系统/网络适配器支持哪种ip版本:
``` C#

       Debug.Log(Socket.OSSupportsIPv4);//.net平台过高       
       Debug.Log(Socket.OSSupportsIPv6);//.net平台过高       
       Debug.Log(Socket.SupportsIPv4);       
       Debug.Log(Socket.SupportsIPv6);//微软标记过时api
 ```
我现在使用的Unity(5.3.4.f1)中mono使用的.net仍然是2.0.50727.1433(Environment.Version),第一和第二条按照msdn说明都是基于现有.net平台(.net4.5+),在unity中执行中肯定会异常,但是在调用的时候发现第一条异常,第二条执行正常,仔细查找mono兼容api发现:
[![](https://hiramtan.files.wordpress.com/2017/05/4442.png?w=809)](https://hiramtan.files.wordpress.com/2017/05/4442.png?w=809)

unity对第二第三第四都提供支持,唯独不支持第一条.第四条被标记成过时api,下面只说明第二第三条.
> **Tip:** 关于stackoverfollow中有人测试说第二条在android上测试异常看来是谬传了.
[![](https://hiramtan.files.wordpress.com/2017/05/55553.png)](https://hiramtan.files.wordpress.com/2017/05/55553.png)

按照接口声明,第二条和第三条在unity中正常使用,并非在android上抛出异常.
再说在unity中支持ipv6,官方说明:
[![](https://hiramtan.files.wordpress.com/2017/05/6662.jpg)](https://hiramtan.files.wordpress.com/2017/05/6662.jpg)

说的很明确,推荐域名,然后通过addressfamily选择合适的ipv4或ipv6,下面就通过tcpclient具体处理ipv6支持.
``` c#
            if (Socket.OSSupportsIPv6)
                client = new TcpClient(AddressFamily.InterNetworkV6);
            else
                client = new TcpClient(AddressFamily.InterNetwork);
```


###
加密说明：
-------------
源码中提供aes加密，需要传入32位密钥，消息传输过程中加密解密会消耗性能，如果对保密要求不严格可以使用简单的移位实现。


support:hiramtan@live.com
***********
MIT License

Copyright (c) [2017] [Hiram]

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.



