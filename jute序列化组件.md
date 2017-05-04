# jute序列化组件
&emsp;&emsp;ZooKeeper的客户端和服务端之间进行一系列的网络通信益实现数据的传输。对于一个网络通信，首先需要解决的就是对数据的序列化和反序列化处理，在ZK中，使用了Jute这一序列化组件来进行数据的序列化和反序列化操作。
### jute对实体类的对象进行序列化和反序列化，大体分为4步：
1. 实体类需要实现Record接口的serialize和deserialize方法。
2. 构建一个序列化器BinaryOutputArchive。
3. 序列化。<br>
调用实体类的serialize方法，将对象序列化到指定tag中去。
4. 反序列化。<br>
调用实体类的deserialize，从指定的tag中反序列化出数据内容。
### 深入Jute
通过Record序列化接口，序列化器和Jute配置文件三方面深入了解Jute
#### Jute
Jute定义了自己独特的序列化格式Record，zookeeper中所有需要进行网络传输或是本地磁盘存储的类型定义，都实现了该接口。<br>
Record接口定义了两个最基本的方法，分别是serialize和deserialize，分别用于序列化和反序列化。其中archive是底层真正的序列化器和反序列化器。并且每个archive中可以包含对多个对象的序列化和反序列化，因此两个接口方法中都标记了参数tag，用于向序列化器和反序列化器标识对象自己的标记。<br>
##### OutputArchive和InputArchive
二者分别是Jute底层的序列化器和反序列化器接口定义。总共有BinaryOutputArchive/BinaryInputArchive、CsvOutputArchive/CsvvInputArchive和XmlOutputArchive/XmlInputArchive三种实现，都是基于OutputStream和InputStream进行操作。<br>
- BinaryInputArchive主要用于进行网络传输和本地磁盘的存储。
- CsvOutputArchive更多的是方便数据对象的可视化展现，因此被使用在toString方法中。
- XmlOutputArchive则是为了将数据对象以XML格式保存和还原，但是目前在ZK中基本没有被用到。
