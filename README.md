# 《深度剖析Apache Dubbo技术内幕》
作者：翟陆续（加多），某大型互联网公司资深开发工程师，并发编程网编辑；热衷并发编程，微服务架构设计，中间件基础设施，著作《Java并发编程之美》,微信公众号：技术原始积累。
# 0.1前言


在单体应用时，不同业务模块部署在同一个JVM进程内，这时候通过本地调用就可以解决不同业务模块之间的相互引用；但多体应用时，不同业务模块大多部署到不同机器上，这时候一 个高效、稳定的RPC框架就显得特别重要了。Apache Dubbo作为阿里巴巴开源的分布式RPC框架，其已进入了Apache 孵化器项目，相信在开源社区的不断贡献下，其会成为RPC框架中的佼佼者。

本书通过原理与实践相结合、由浅入深、通俗易懂的方式讲解了Dubbo框架如何使用与内核原理实现，详细读者阅读完本书后，对一个RPC框架应该具有那些模块，以及各个模块之间如何组合起来的，有深入的理解；本书适合Java中高级研发工程师，以及对RPC框架技术感兴趣，希望探究RPC框架内部实现原理的人员阅读。

##  0.2为何要研究Apache Dubbo的实现原理
引用我在 [《Java并发编程之美》](https://item.jd.com/12450812.html)一书前言中的论述：研究开源框架、特别是优秀的开源框架的实现原理，可以开拓我们的技术视野，提高我们的架构能力，减少由于使用不当导致的线上故障的发生。而在微服务大行其道的今天，RPC框架作为微服务之间通信的一种手段，其在微服务架构中占有一席之地，Apache Dubbo则作为RPC框架中比较优秀的代表，为了更好的使用它，其实现原理自然值得我们去探究。

下面我们来具体谈谈研究Dubbo框架实现原理，我们到底能学到什么？

首先我们可以学习和深刻体会到分层架构带来的好处，Dubbo框架在整体是分为了业务层(Business)、RPC层、远程调用(Remoting)层；其中业务层提供API让使用者方便的发布与引用服务；RPC层则是对服务注册与发现、服务代理、路由、负载均衡等功能的封装，该层又有可以被划分为好多层；远程调用层则是对网络传输与请求数据序列/反序列化等的抽象；使用分层架构可以保证下层的改变对上层不可见，并且可以实现关注点分离，比如使用者使用Dubbo时候只关心如何使用业务层的API来发布与引用服务，而不需要关心RPC层的实现，当新版Dubbo升级了RPC层的逻辑时候，使用方只需要升级Dubbo的版本就可以了，这是因为RPC层的修改对业务层使用者来着是透明的。

我们也可以学习到好的框架应该具有可扩展性，Dubbo就是一个扩展性极强的框架，其RPC层中的所有组件都是基于SPI扩展接口实现的，每个组件都是可以被替换的；Dubbo 增强了 JDK 中提供的标准 SPI 功能，并且增加了对扩展接口的IoC （一个扩展接口可以直接 setter 注入其它扩展接口）和 AOP 的支持（可以使用Wrapper类对扩展接口进行功能增强）；并且增强SPI不会一次性实例化扩展点的所有实现类，这避免了当扩展点实现类初始化很耗时，但当前还没用上它的功能时仍进行加载实例化，浪费资源的情况；增强的 SPI 是在具体用某一个实现类的时候才对具体实现类进行实例化。

作为高可用分布式RPC框架，其自身必须具有容错能力，以便提高系统的可用性；Dubbo框架则提供了分布式系统中常见的集群容错策略，并且提供了扩展接口，让使用方方便的定制自己的集群容错策略，通过研究Dubbo框架提供的集群容错策略，可以让我们对分布式系统中的容错技术有深入的理解。

在分布式系统中每个微服务都是以集群的方式部署，那么当我们访问一个具体服务时候到底访问那一台机器提供的服务那？这就是分布式系统中负载均衡器与路由规则要做的事情，作为分布式RPC框架，其自身也必须具有负载均衡的能力，Dubbo框架则提供了分布式系统中常见的负载均衡策略，并且提供了扩展接口，让使用方方便的定制自己的负载均衡策略；另外路由规则提供了服务治理的一种策略，在Dubbo中我们可以通过管理控制台来配置路由规则，让消费者只可访问那些服务提供者；通过研究Dubbo框架提供的负载均衡与路由策略，可以让我们对分布式系统中的负载均衡技术与路由规则有深入的理解。

在分布式系统中当我们要消费某个服务时候，如何找到其地址是一个要解决的问题，在分布式RPC中一个通用解决方案是引入服务注册中心，当服务提供者启动时候会自动把自己的服务注册到服务注册中心，当消费者启动时候会去服务注册中心订阅自己感兴趣的服务的地址列表；在Dubbo框架中则提供了扩展接口来方便的让我们使用zookeeper、redis等作为服务注册中心，通过研究Dubbo原理，我们可以深刻理解服务提供方到底是如何把服务注册到服务注册中心的，以及服务消费端如何动态的感知服务提供方地址列表变化的。

所有RPC框架都要解决的一个问题是，如何让使用者无感知的发起远程过程调用，也就是让使用者在发起远程调用时候，有和本地调用一样的体验；Dubbo框架和其他RPC框架一样采用代理来实现该能力，在Dubbo框架中扩展接口Proxy就是专门来做代理使用的，并且其提供了扩展接口的JDK动态代理与Cglib的实现；研究Dubbo的原理，我们可以学习到消费端如何对服务接口进行的代理以实现透明调用，服务提供端如何使用代理与JavaAssist技术减少反射调用开销的。

在Dubbo的分层架构中Transport 网络传输层把抽象 mina 和 netty 为统一接口，并且默认情况下其使用Netty作为底层网络通信，通过研究Dubbo，我们可以学习到Dubbo的网络协议帧是如何设计的；服务消费端如何启动Netty客户端的，如何把rpc请求封装为协议帧并序列化，然后通过Netty客户端发起网络请求的；服务提供端又是如何启动Netty服务器进行服务监听的，如何处理经典的半包、粘包问题的，如何把接受到的二进制包转换为Dubbo协议帧，并反序列化为POJO对象的；另外使用Netty时候都说不要在ChannelHandler中做阻塞的事情，以免阻塞了IO线程，使其他请求得不到及时处理，那么这到底是什么意思那？研究完Dubbo的线程模型你就会明白了。

对于网络请求来说，同步调用时比较直截了当的，但是同步调用意味着当前发起请求的调用线程在远端机器返回结果前必须阻塞等待，这明显很浪费资源。好的做法应该是发起请求的调用线程发起请求后，注册一个回调函数，然后马上返回去做其他事情，当远端把结果返回后在使用IO线程执行回调函数，也就是发起方实现了异步调用，调用线程不会被阻塞。Dubbo则基于Netty的异步非阻塞能力与JDK8中的CompletableFuture轻松的实现RPC请求的异步调用，提高了资源利用率；通过研究Dubbo的实现原理，我们可以对异步编程带来的好处以及实现原理有深刻的体会。

...

总之研究透彻Dubbo框架原理实现后，你会对分布式系统中的很多技术点有深入的理解；而我坚信分布式系统是应用的发展方向，因为随着业务规模的增大，为了保障系统的可伸缩性、高可用性，系统必然朝着分布式方向发展；所以如果能掌握一些分布式系统中的优秀RPC框架的原理以及实现细节，无论是现在还是将来都将会成为自己区别于他人的核心竞争力。


## 0.3如何阅读本书
本书分为三部分:
- 第一部分为基础篇，首先从整体讲解使用Dubbo搭建的系统都有哪些模块组成，其相互之间调用关系是怎么样的，然后基于本书Demo讲解如何使用Dubbo；
- 第二部分高级篇主要讲解Dubbo框架内部实现原理，包含支撑Dubbo框架的适配器类原理、动态编译原理、增强SPI实现原理、消费端的泛化调用实现原理、消费端异步调用与服务提供端的异步执行、Dubbo框架的线程模型、消费端负载均衡策略、消费端集群容错策略、并发控制原理、Dubbo网络协议等等；
- 第三部分实践篇主要探讨如何使用Arthas和一些demo来为研究Dubbo框架原理提供便捷，并且讲解如何基于Netty与CompletableFuture模拟RPC同步与纯异步调用。


读者可以在[https://github.com/zhailuxu/Dubbo-Demo](https://github.com/zhailuxu/Dubbo-Demo)
下载本书的demo的源码；另外由于作者时间与能力有限，书中的内容难免有错误发生，如果读者在阅读本书时发现错误，可以加我微信公众号：技术原始积累，反馈。

## 0.4 阿里巴巴高级技术专家 Apache Dubbo PMC Chair ：罗毅（北纬) 推荐序
自从我的团队开始维护 Dubbo 开源，到今年5月21日Dubbo从Apache中毕业，成为又一个国人主导的顶级开源项目，时间大概经历了接近两年的时间。在毕业之际，Dubbo开源团队收到了来自社区的很多朋友的祝贺。有朋友过来恭喜我这个项目开源成功了，感谢之余我的回复是Dubbo开源才刚刚渐入佳境，谈不上成功，这一点，在媒体对Dubbo毕业的采访中我是这么表述的

“从 Apache 毕业，对 Dubbo 而言，是一件里程碑的事件，对 The Apache Way 而言，也是一件非常有意义的事情。Dubbo 捐献给 Apache 软件基金会并开始孵化那时候，参与社区贡献的人并不多，但今天 Dubbo 的贡献者数量增加了近5倍，我们为此感到自豪。我很荣幸自己能参与其中，我们的旅程将继续，相信开源社区将使 Apache Dubbo 更加强大。”

毕业不代表结束，而是新的开始。那么一个开源项目怎样才算成功呢？在我看来，生产环境中实践的越多，参与贡献的人越多，愿意学习掌握的人越多，就越成功。而帮助更多的人了解Dubbo，掌握Dubbo，学习资料的丰富程度就成为了关键。这不是靠Dubbo开源团队的几篇博文，实例代码，和官方文档能够撑得起来的。我是非常的期望看到有更多的Dubbo的出版物出现。

所幸的是，在这两年中，我的确是看到了越来越多的科技博主在介绍Dubbo，有人开始在知识付费频道讲授Dubbo的课程，我本人也有幸被几本书的作者邀请写推荐或者序。今天，又有幸被我在阿里的同事翟陆续邀请，为他的新书《深度剖析Apache Dubbo技术内幕》做序。翟陆续，花名加多，担任并发编程网编辑，同时也是另一本书 [《Java并发编程之美》](https://item.jd.com/12450812.html) 的作者。他带来的这本书是基于Dubbo目前的主干版本2.7。作者深入浅出的从基本用法说起，然后逐一剖析了Dubbo的架构、扩展、服务的注册与发现、路由、集群、线程、协议与网络等方方面面，基本上涵盖了Dubbo这个服务框架最主要的部分。对于想深入到Dubbo源码的读者有着相当大的帮助。最后，作者还介绍了如何使用Arthas来帮助理解Dubbo的工作，如何利用Dubbo的SPI扩展机制来实现自定义的负载均衡策略。作者在一本书中介绍了两个来自我的团队的开源作品，也让我有点受宠若惊。

最后，也通过写这个序的机会，感谢一下这本书的作者加多同学。这本书是目前市面上能够看到的唯一一本基于2.7介绍Dubbo原理的书籍。作者通过关键源码、与图文的有机结合，深入浅出的介绍了Dubbo的关键实现。也期待更多的开源爱好者可以通过学习这本书的同时，开始加入到Dubbo开源的大家庭里来，把国人主导的这个Apache顶级项目做的越来越好。


## 0.5 购买链接
- [京东](https://item.jd.com/12769688.html?dist=jd)
- 淘宝



