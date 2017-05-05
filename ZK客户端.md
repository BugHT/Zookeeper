ZK客户端
=====
客户端主要核心组件：
- Zookeeper实例：客户端入口
- ClientWatcherManager：客户端Watcher管理器
- HostProvider：客户端地址列表管理器
- ClientCnxn：客户端核心线程，其内部又包含两个线程，即SendThread和EventThread。前者是一个I/O线程，主要负责ZK客户端和服务端之间的网络I/O通信，后者是一个事件线程，主要负责对服务端时间进行处理。
<br><br>
客户端的整个初始化和启动过程大体可以分为3个步骤
1. 设置默认Watcher。
2. 设置Zookeeper服务器地址列表
3. 创建ClientCnxn。
<br>
如果在Zookeeper的构造方法中传入一个Watcher对象的话，那么Zookeeper就会将这个Watcher对象保存在ZKWatcherManager的defaultWatcher中，作为整个客户端会话期间的默认Watcher。

SendThread是客户端ClientCnxn内部的一个核心I/O调度进程，用于管理客户端与服务端之间的所有网络I/O操作，具体作用：
1. 维护了客户端与服务端之间的会话生命周期，通过一定周期频率内向服务端发送PING包检测心跳，如果会话周期内客户端与服务端出现TCP连接断开，那么就会自动且透明的完成重连操作。
2. 管理了客户端所有的请求发送和响应接受操作，其将上层客户端API操作转换成相应的请求协议并发送到服务端，并完成对同步调用的返回和异步调用的回调。
3. 将来自服务端的事件传递给EventThread去处理。

EventThread是客户端ClientCnxn内部的一个事件处理线程，负责客户端的事件处理，并触发客户端注册的Watcher监听，EventThread中的watingEvents队列用于临时存放那些需要被触发的Object，包括客户端注册的Watcher和异步接口中注册的回调器AsnycCallback。同时，EVentThread会不断地从watingEvents中取出Object，识别具体类型（Watcher或AsnycCallback），并分别调用process和processResult接口方法来实现对事件的触发和回调。

### 客户端和服务端的长连接的session失效，超时，连接断掉的问题以及处理方式
ClientCnxn包括以下几个变量：
- private int connectTimeout；
- private volatile int negotiatedSessionTimeout；
- private int readTimeout；
- private final int sessionTimeout；
在client链接到server后，server返回给client确认信息（包括服务器返回给客户端的真实的timeout时间--negotiatedSessionTimeout）
