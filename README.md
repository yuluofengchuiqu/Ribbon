# Ribbon

1.Ribbon的大体流程
一般我们在使用Spring Cloud框架的时候并不知道Ribbon是怎么使用的，那么如果我们想要去研究它，应该从什么地方入手呢？

如果在没有使用Feign调用接口的时候，我们肯定会选择RestTemplate来调用，Ribbon给提供了一个负载均衡的一个标注@LoadBalanced，可以加在RestTemplate上，这个就是我们接下来要看Ribbon的入口了。

@LoadBalanced底层其实就是个拦截器，拦截了所有的RestTemplate调用的接口，在通过调用的是哪个服务来判断对应的ip，拼接好正确的url之后，在通过HTTP请求再去请求对应的接口就可以了。

Ribbon的核心组件有，ILoadBalancer、IRule和IPing。

2.核心入口类
如果大家想找的话，会比较麻烦，可能找了很多很多都没有找到，我在这里直接告诉大家，RibbonLoadBalancerClient，所在的jar包是spring-cloud-netflix-robbin-2.x.x.RELEASE.jar。

3.核心组件ILoadBalancer
大家不用猜也知道，这个ILoadBalancer 肯定是一个接口，肯定会有很多的实现

![](https://raw.githubusercontent.com/yuluofengchuiqu/Ribbon/main/img/20210126194829782.png)

这个就是ILoadBalancer的默认实现ZoneAwareLoadBalancer。

![](https://raw.githubusercontent.com/yuluofengchuiqu/Ribbon/main/img/99247780566a698b545f143daba45ad7.png)

![](https://raw.githubusercontent.com/yuluofengchuiqu/Ribbon/main/img/20210126194858493.png)

上面两张图，可以看到ZoneAwareLoadBalancer的继承关系。

其实选择相对来说好说一些，重点是这个里面的服务集合是怎么存的呢？这个可能就需要看到DynamicServerListLoadBalancer里面的restOfInit和updateListOfServers来操作对应的集合，集合的数据保存到BaseLoadBalancer里面。这个里面的服务集合就是根据所在的Eureka Client来获取的，这里又到了怎么拉取的工作了，默认是每30秒拉取一次的，保证和Eureka Server尽可能的保持相同的数据。

怎么选择对应的服务实例的呢？大家看看ZoneAwareLoadBalancer里面有个chooseServer方法，这个就是选择服务实例的方法，但是最后还是走的是BaseLoadBalancer里面的chooseServer方法。

![](https://raw.githubusercontent.com/yuluofengchuiqu/Ribbon/main/img/20210126194936507.png)

![](https://raw.githubusercontent.com/yuluofengchuiqu/Ribbon/main/img/17d8b2166030c7694c5f3421b906c90a.png)

看到这里还是不知道怎么从服务实例的集合里面选择出来一个，接下来就要看Ribbon的另外一个组件IRule了。

4.核心组件IRule
其实IRule并没有什么可以说的地方，了解一下代码里面呆的几个负责均衡就可以。

RoundRobinRule：系统内置的默认负载均衡规范，直接round robin轮询，从一堆server list中，不断的轮询选择出来一个server，每个server平摊到的这个请求，基本上是平均的AvailabilityFilteringRule：这个rule就是会考察服务器的可用性

WeightedResponseTimeRule：带着权重的，每个服务器可以有权重，权重越高优先访问，如果某个服务器响应时间比较长，那么权重就会降低，减少访问

ZoneAvoidanceRule：根据机房和服务器来进行负载均衡，说白了，就是机房的意思，看了源码就是知道了，这个就是所谓的spring cloud ribbon环境中的默认的Rule

BestAvailableRule：忽略那些请求失败的服务器，然后尽量找并发比较低的服务器来请求

RandomRule：随机找一个服务器，尽量将流量分散在各个服务器上

RetryRule：可以重试，就是通过round robin找到的服务器请求失败，可以重新找一个服务器

5.核心组件IPing
IPing直接按字面理解就好了，就是看看服务是否还可以连接。