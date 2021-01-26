## 一个非常小的rpc 样品，基于java8工具包实现。
 RPC是远程过程调用协议，其作用就是客户端与服务端之间的远程调用，就像本地自己调用一样，让服务端进行服务化，功能唯一性，负载热点流量。

RPC能够让本地应用简单、高效地调用服务器中的过程（服务）。它主要应用在分布式系统。如Hadoop中的IPC组件。一个RPC工具，要考虑如下几个点：

- 通信模型：假设通信的为A机器与B机器，A与B之间有通信模型，在Java中一般基于BIO或NIO；。
- 过程（服务）定位：使用给定的通信方式，与确定IP与端口及方法名称确定具体的过程或方法；
- 远程代理对象：本地调用的方法(服务)其实是远程方法的本地代理，因此可能需要一个远程代理对象，对于Java而言，远程代理对象可以使用Java的动态对象实现，封装了调用远程方法调用；
- 序列化，将对象名称、方法名称、参数等对象信息进行网络传输需要转换成二进制传输，这里可能需要不同的序列化技术方案。如:thrift,protobuf,Arvo等。

 

本仓库是一个基于java8 socket来实现的rpc调用示例，以加深对rpc实现的本质认识。

服务协议接口：

```java
package top.tiny.group.proto;

public interface IHello {
      String sayHello(String string);
}
```



服务协议接口实现：

```java
package top.tiny.group.proto.impl;

import top.tiny.group.proto.IHello;

public class HelloServiceImpl implements IHello {

    @Override
    public String sayHello(String string) {
        return "rpc response："+string;
    }
}
```



RpcClient实现：

```java
package top.tiny.group.remoting;

import top.tiny.group.proto.IHello;

import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.net.InetSocketAddress;
import java.net.Socket;

public class RpcClientProxy<T> implements InvocationHandler {

    private  Class<T> serviceInterface;
    private InetSocketAddress addr;

    public RpcClientProxy(Class<T> serviceInterface, String ip,String port) {
        this.serviceInterface = serviceInterface;
        this.addr = new InetSocketAddress(ip, Integer.parseInt ( port ));
    }

    public T getClientIntance(){
        return (T) Proxy.newProxyInstance (serviceInterface.getClassLoader(),new Class<?>[]{serviceInterface},this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        Socket socket = null;
        ObjectOutputStream output = null;
        ObjectInputStream input = null;

        try {
            // 2.创建Socket客户端，根据指定地址连接远程服务提供者
            socket = new Socket();
            socket.connect(addr);

            // 3.将远程服务调用所需的接口类、方法名、参数列表等编码后发送给服务提供者
            output = new ObjectOutputStream(socket.getOutputStream());
            output.writeUTF(serviceInterface.getName());
            output.writeUTF(method.getName());
            output.writeObject(method.getParameterTypes());
            output.writeObject(args);

            // 4.同步阻塞等待服务器返回应答，获取应答后返回
            input = new ObjectInputStream(socket.getInputStream());
            return input.readObject();
        } finally {
            if (socket != null) socket.close();
            if (output != null) output.close();
            if (input != null) input.close();
        }
    }

    public static void main(String[] args) {
        RpcClientProxy client = new RpcClientProxy<>(IHello.class,"localhost","6666");
        IHello hello = (IHello) client.getClientIntance ();
        System.out.println (hello.sayHello ( "socket rpc" ));
    }
}
```



RpcServer实现：

```java
package top.tiny.group.remoting;

import top.tiny.group.proto.IHello;
import top.tiny.group.proto.impl.HelloServiceImpl;

import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Method;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.HashMap;

public class RpcServer {

    private static final HashMap<String, Class<?>> serviceRegistry = new HashMap<>();
    private  int port;

    public RpcServer(int port) {
        this.port =port;
    }

    public RpcServer register(Class serviceInterface, Class impl) {
        serviceRegistry.put(serviceInterface.getName(), impl);
        return this;
    }

    public void run() throws IOException {

        ServerSocket server = new ServerSocket();
        server.bind(new InetSocketAddress(port));
        System.out.println("start server");
        ObjectInputStream input =null;
        ObjectOutputStream output =null;
        Socket socket=null;
        try {
            while(true){
                socket = server.accept ();
                input =new ObjectInputStream(socket.getInputStream());
                String serviceName = input.readUTF();
                String methodName = input.readUTF();
                System.out.println (methodName);
                Class<?>[] parameterTypes = (Class<?>[]) input.readObject();
                Object[] arguments = (Object[]) input.readObject();
                Class serviceClass = serviceRegistry.get(serviceName);
                if (serviceClass == null) {
                    throw new ClassNotFoundException(serviceName + " not found");
                }
                Method method = serviceClass.getMethod(methodName, parameterTypes);
                Object result = method.invoke(serviceClass.newInstance(), arguments);
                output = new ObjectOutputStream (socket.getOutputStream());
                output.writeObject(result);
            }
        } catch (Exception e){
            e.printStackTrace();
        }finally {
            if (output != null) {
                try {
                    output.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (input != null) {
                try {
                    input.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (socket != null) {
                try {
                    socket.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }

    }

    public static void main(String[] args) throws IOException {
        new RpcServer ( 6666 ).register ( IHello.class, HelloServiceImpl.class).run ();
    }

}
```

上面即是一个最简单化的Rpc实现，有如下几点比较明显的局限性：

- 序列化局限，原生序列化只能序列化实现了【Serializable】接口的服务类，并且序列化复杂对象时，内容庞大效率极低，需要高效的序列化协议进行序列化参数方法等必要请求入参
- BIO性能局限，socket服务端采用默认的BIO来阻塞获取输入流，效率低下，需采用NIO等异步非阻塞服务端方案，例如netty,mina和java nio等。
- 在大型企业级RPC解决方案中，客户端和服务端的长连接需要一直保持，否则每次调用时都要重新进行三次握手和四次挥手，这样频繁的创建tcp连接对机器性能是极大的损耗，对socket的连接可以采用apache pool2连接池等方案
- 服务端负载，需要考虑服务自动发现，让客户端在不需要重启的情况下能动态感知服务端的变化，从而实现热部署等。可以采用办法定时自动轮询，zookeeper等。
- 服务端服务类执行异常，客户端感知等。

基于这个最简单的rpc认识，可以开始进一步阅读 基于zk、thrift、netty的企业级RPC解决方案。



参考：https://www.oschina.net/group/ai-bigdata#/detail/1510027
