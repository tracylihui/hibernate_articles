title: 'Hibernate：缓存机制的学习'
date: 2015-07-20 14:47:52
categories: Hibernate
tags: [Hibernate,缓存]
---
Hibernate中会经常用到set等集合来表示1-N的关系。比如，我有Customer和Order两个对象。其中，在Customer中有一个Order的set集合，表示在一个顾客可以拥有多个Order，而在Order对象中存在了一个Customer的对象，表示这个Order是哪个顾客下的单。这个算是比较典型的双向1-N关联。

这给我们带来了很大的好处，当我得到了Customer对象的时候，我们可以很方便的将与其相关联的Order集合查询出来，这也非常符合我们的实际业务，毕竟我们不可能给这个Cutomer对象别人的Order吧，这既不安全，而且对Customer的普通顾客来说，并无卵用。所以我们不得不说Hibernate的ORM做的很好，但凡事都有但是（要是没有但是，也就没有写这篇文章的必要了）。

我们再对数据库进行访问的时候必须要考虑性能问题（通俗点讲，就是用少发SQL语句），当我们设定了1-N这种关系后，查询过程中就有可能出现N+1问题。

关于N+1问题，并不是本文的重点。但关于N+1问题，我们需要知道的是，这个问题会导致SQL语句的增加，也就是要与数据库进行更多的交互，这无疑会给项目以及后台数据库带来影响。
<!--more-->

#Hibernate缓存
---
Hibernate是一个持久化框架，经常需要访问数据库。如果我们能够降低应用程序对物理数据库访问的频次，那会提供应用程序的运行性能。缓存内的数据是对物理数据源中的数据的复制，应用程序运行时先从缓存中读写数据。

缓存就是数据库数据在内存中的临时容器，包括数据库数据在内存中的临时拷贝，它位于数据库与数据库访问层中间。ORM在查询数据时首先会根据自身的缓存管理策略，在缓存中查找相关数据，如发现所需的数据，则直接将此数据作为结果加以利用，从而避免了数据库调用性能的开销。而相对内存操作而言，数据库调用是一个代价高昂的过程。

Hibernate缓存包括两大类：一级缓存和二级缓存。

- Hibernate一级缓存又被成为“Session的缓存”。Session缓存是内置的，不能被卸载，是事务范围的缓存。在一级缓存中，持久化类的每个实例都具有唯一的OID。
- Hibernate二级缓存又被称为“SessionFactory的缓存”。由于SessionFactory对象的生命周期和应用程序的整个过程对应，因此Hibernate二级缓存是进程范围或者集群范围的缓存，有可能出现并发问题，因此需要采用适当的并发访问策略，该策略为被缓存的数据提供了事务隔离级别。第二级缓存是可选的，是一个可配置的插件，默认下SessionFactory不会启用这个插件。

那么什么样的数据适合放入到缓存中？
- 很少被修改的数据 　　
- 不是很重要的数据，允许出现偶尔并发的数据 　　
- 不会被并发访问的数据 　　
- 常量数据

什么样的数据不适合放入到缓存中？　
- 经常被修改的数据 　　
- 绝对不允许出现并发访问的数据，如财务数据，绝对不允许出现并发 　　
- 与其他应用共享的数据

#Hibernate一级缓存
---
##Demo
首先看一个非常简单的例子：
```java
@Test
public void test() {
	Customer customer1 = (Customer) session.load(Customer.class, 1);
	System.out.println(customer1.getCustomerName());

	Customer customer2 = (Customer) session.load(Customer.class, 1);
	System.out.println(customer2.getCustomerName());
}
```
看一下控制台的输出：
```
Hibernate:
    select
        customer0_.CUSTOMER_ID as CUSTOMER1_0_0_,
        customer0_.CUSTOMER_NAME as CUSTOMER2_0_0_
    from
        CUSTOMERS customer0_
    where
        customer0_.CUSTOMER_ID=?
Customer1
Customer1
```
我们可以看到，虽然我们调用了两次session的load方法，但实际上只发送了一条SQL语句。我们第一次调用load方法时候，得到了查询结果，然后将结果放到了session的一级缓存中。此时，当我们再次调用load方法，会首先去看缓存中是否存在该对象，如果存在，则直接从缓存中取出，就不会在发送SQL语句了。

但是，我们看一下下面这个例子：
```java
@Test
public void test() {
	Customer customer1 = (Customer) session.load(Customer.class, 1);
	System.out.println(customer1.getCustomerName());
	transaction.commit();
	session.close();
	session = sessionFactory.openSession();
	transaction = session.beginTransaction();
	Customer customer2 = (Customer) session.load(Customer.class, 1);
	System.out.println(customer2.getCustomerName());
}
```
我们解释一下上面的代码，在第5、6、7、8行，我们是先将session关闭，然后又重新打开了新的session，这个时候，我们再看一下控制台的输出结果：
```
Hibernate:
    select
        customer0_.CUSTOMER_ID as CUSTOMER1_0_0_,
        customer0_.CUSTOMER_NAME as CUSTOMER2_0_0_
    from
        CUSTOMERS customer0_
    where
        customer0_.CUSTOMER_ID=?
Customer1
Hibernate:
    select
        customer0_.CUSTOMER_ID as CUSTOMER1_0_0_,
        customer0_.CUSTOMER_NAME as CUSTOMER2_0_0_
    from
        CUSTOMERS customer0_
    where
        customer0_.CUSTOMER_ID=?
Customer1
```
我们可以看到，发送了两条SQL语句。其原因是：Hibernate一级缓存是session级别的，所以如果session关闭后，缓存就没了，当我们再次打开session的时候，缓存中是没有了之前查询的对象的，所以会再次发送SQL语句。

我们稍微对一级缓存的知识点进行总结一下，然后再开始讨论关于二级缓存的内容。
##作用
Session的缓存有三大作用：
1. 减少访问数据库的频率。应用程序从缓存中读取持久化对象的速度显然比到数据中查询数据的速度快多了，因此Session的缓存可以提高数据访问的性能。
2. 当缓存中的持久化对象之间存在循环关联关系时，Session会保证不出现访问对象图的死循环，以及由死循环引起的JVM堆栈溢出异常。
3. 保证数据库中的相关记录与缓存中的相应对象保持同步。

##小结
- 一级缓存是事务级别的，每个事务(session)都有单独的一级缓存。这一级别的缓存是由Hibernate进行管理，一般情况下无需进行干预。
- 每个事务都拥有单独的一级缓存不会出现并发问题，因此无须提供并发访问策略。
- 当应用程序调用Session的save()、update()、saveOrUpdate()、get()或load()，以及调用查询接口的 list()、iterate()(该方法会出现N+1问题，先查id)方法时，如果在Session缓存中还不存在相应的对象，Hibernate就会把该对象加入到第一级缓存中。当清理缓存时，Hibernate会根据缓存中对象的状态变化来同步更新数据库。 Session为应用程序提供了两个管理缓存的方法： evict(Object obj)：从缓存中清除参数指定的持久化对象。 clear()：清空缓存中所有持久化对象,flush():使缓存与数据库同步。
- 当查询相应的字段，而不是对象时，不支持缓存。我们可以很容易举一个例子来说明，看一下下面的代码。

```java
@Test
public void test() {
	List<Customer> customers = session.createQuery("select c.customerName from Customer c").list();
	System.out.println(customers.size());
	Customer customer2 = (Customer) session.load(Customer.class, 1);
	System.out.println(customer2.getCustomerName());
}
```
我们首先是只取出Customer的name属性，然后又尝试着去Load一个Customer对象，看一下控制台的输出：
```
Hibernate:
    select
        customer0_.CUSTOMER_NAME as col_0_0_
    from
        CUSTOMERS customer0_
3
Hibernate:
    select
        customer0_.CUSTOMER_ID as CUSTOMER1_0_0_,
        customer0_.CUSTOMER_NAME as CUSTOMER2_0_0_
    from
        CUSTOMERS customer0_
    where
        customer0_.CUSTOMER_ID=?
Customer1
```
这一点其实很好理解，我本身就没有查处Customer的所有属性，那我又怎么能给你把所有属性都缓存到这个对象中呢？

我们在讲之前的例子中，提到我们关闭session再打开，这个时候一级缓存就不存在了，所以我们再次查询的时候，会再次发送SQL语句。那么如果要解决这个问题，我们该怎么做？二级缓存可以帮我们解决这个问题。

#Hibernate二级缓存
Hibernate中没有自己去实现二级缓存，而是利用第三方的。简单叙述一下配置过程，也作为自己以后用到的时候配置的一个参考。

1、我们需要加入额外的二级缓存包，例如EHcache，将其包导入。需要：ehcache-core-2.4.3.jar ， hibernate-ehcache-4.2.4.Final.jar ，slf4j-api-1.6.1.jar
2、在hibernate.cfg.xml配置文件中配置我们二级缓存的一些属性（此处针对的是Hibernate4）：
```xml
<!-- 启用二级缓存 -->
<property name="cache.use_second_level_cache">true</property>
<!-- 配置使用的二级缓存的产品 -->
<property name="hibernate.cache.region.factory_class">org.hibernate.cache.ehcache.EhCacheRegionFactory</property>
```
3、我们使用的是EHcache，所以我们需要创建一个ehcache.xml的配置文件，来配置我们的缓存信息，这个是EHcache要求的。该文件放到根目录下。
```xml
<ehcache>
    <!--  
    	指定一个目录：当 EHCache 把数据写到硬盘上时, 将把数据写到这个目录下.
    -->
    <diskStore path="d:\\tempDirectory"/>
    <!--Default Cache configuration. These will applied to caches programmatically created through
        the CacheManager.
        The following attributes are required for defaultCache:
        maxInMemory       - Sets the maximum number of objects that will be created in memory
        eternal           - Sets whether elements are eternal. If eternal,  timeouts are ignored and the element
                            is never expired.
        timeToIdleSeconds - Sets the time to idle for an element before it expires. Is only used
                            if the element is not eternal. Idle time is now - last accessed time
        timeToLiveSeconds - Sets the time to live for an element before it expires. Is only used
                            if the element is not eternal. TTL is now - creation time
        overflowToDisk    - Sets whether elements can overflow to disk when the in-memory cache
                            has reached the maxInMemory limit.
        -->
    <!--  
    	设置缓存的默认数据过期策略
    -->
    <defaultCache
        maxElementsInMemory="10000"
        eternal="false"
        timeToIdleSeconds="120"
        timeToLiveSeconds="120"
        overflowToDisk="true"
        />
   	<!--  
   		设定具体的命名缓存的数据过期策略。每个命名缓存代表一个缓存区域
   		缓存区域(region)：一个具有名称的缓存块，可以给每一个缓存块设置不同的缓存策略。
   		如果没有设置任何的缓存区域，则所有被缓存的对象，都将使用默认的缓存策略。即：<defaultCache.../>
   		Hibernate 在不同的缓存区域保存不同的类/集合。
			对于类而言，区域的名称是类名。如:com.atguigu.domain.Customer
			对于集合而言，区域的名称是类名加属性名。如com.atguigu.domain.Customer.orders
   	-->
   	<!--  
   		name: 设置缓存的名字,它的取值为类的全限定名或类的集合的名字
		maxElementsInMemory: 设置基于内存的缓存中可存放的对象最大数目

		eternal: 设置对象是否为永久的, true表示永不过期,
		此时将忽略timeToIdleSeconds 和 timeToLiveSeconds属性; 默认值是false
		timeToIdleSeconds:设置对象空闲最长时间,以秒为单位, 超过这个时间,对象过期。
		当对象过期时,EHCache会把它从缓存中清除。如果此值为0,表示对象可以无限期地处于空闲状态。
		timeToLiveSeconds:设置对象生存最长时间,超过这个时间,对象过期。
		如果此值为0,表示对象可以无限期地存在于缓存中. 该属性值必须大于或等于 timeToIdleSeconds 属性值

		overflowToDisk:设置基于内存的缓存中的对象数目达到上限后,是否把溢出的对象写到基于硬盘的缓存中
   	-->
    <cache name="com.atguigu.hibernate.entities.Employee"
        maxElementsInMemory="1"
        eternal="false"
        timeToIdleSeconds="300"
        timeToLiveSeconds="600"
        overflowToDisk="true"
        />

    <cache name="com.atguigu.hibernate.entities.Department.emps"
        maxElementsInMemory="1000"
        eternal="true"
        timeToIdleSeconds="0"
        timeToLiveSeconds="0"
        overflowToDisk="false"
        />

</ehcache>
```
在注释中，有一些对变量的解释。

4、开启二级缓存。我们在这里使用的xml的配置方式，所以要在Customer.hbm.xml文件加一点配置信息：
```xml
<cache usage="read-only"/>
```
注意是在<class>标签内。
如果是使用注解的方法，在要在Customer这个类中，加入`@Cache(usage=CacheConcurrencyStrategy.READ_ONLY)`这个注解。

5、下面我们再进行一下测试。还是上面的代码：
```java
@Test
public void test() {
	Customer customer1 = (Customer) session.load(Customer.class, 1);
	System.out.println(customer1.getCustomerName());
	transaction.commit();
	session.close();
  session = sessionFactory.openSession();
	transaction = session.beginTransaction();
	Customer customer2 = (Customer) session.load(Customer.class, 1);
	System.out.println(customer2.getCustomerName());
}
```
我们可以发现控制台只发出了一条SQL语句。这是我们二级缓存的一个小Demo。

我们的二级缓存是sessionFactory级别的，所以当我们session关闭再打开之后，我们再去查询对象的时候，此时Hibernate会先去二级缓存中查询是否有该对象。

同样，二级缓存缓存的是对象，如果我们查询的是对象的一些属性，则不会加入到缓存中。

**我们通过二级缓存是可以解决之前提到的N+1问题。**

已经写了这么多了，但好像我们关于缓存的内容还没有讲完。不要着急，再坚持一下，我们的内容不多了。我们还是通过一个例子来引出下一个话题。
我们说通过二级缓存可以缓存对象，那么我们看一下下面的代码以及输出结果：
```java
@Test
public void test() {
	List<Customer> customers1 = session.createQuery("from Customer").list();
	System.out.println(customers1.size());
	tansaction.commit();
	session.close();
	session = sessionFactory.openSession();
	transaction = session.beginTransaction();
	List<Customer> customers2 = session.createQuery("from Customer").list();
	System.out.println(customers2.size());
}
```
控制台的结果：
```
Hibernate:
    select
        customer0_.CUSTOMER_ID as CUSTOMER1_0_,
        customer0_.CUSTOMER_NAME as CUSTOMER2_0_
    from
        CUSTOMERS customer0_
3
Hibernate:
    select
        customer0_.CUSTOMER_ID as CUSTOMER1_0_,
        customer0_.CUSTOMER_NAME as CUSTOMER2_0_
    from
        CUSTOMERS customer0_
3
```
我们的缓存好像没有起作用哎？这是为啥？当我们通过list()去查询两次对象的时候，二级缓存虽然会缓存插叙出来的对象，但不会缓存我们的hql查询语句，要想解决这个问题，我们需要用到查询缓存。

#查询缓存
---
在前文中也提到了，我们的一级二级缓存都是对整个实体进行缓存，它不会缓存普通属性，如果想对普通属性进行缓存，则可以考虑使用查询缓存。

但需要注意的是，大部分情况下，查询缓存并不能提高应用程序的性能，甚至反而会降低应用性能，因此实际项目中要谨慎的使用查询缓存。

对于查询缓存来说，它缓存的key就是查询所用的HQL或者SQL语句，需要指出的是：查询缓存不仅要求所使用的HQL、SQL语句相同，甚至要求所传入的参数也相同，Hibernate才能直接从缓存中取得数据。只有经常使用相同的查询语句、并且使用相同查询参数才能通过查询缓存获得好处，查询缓存的生命周期直到属性被修改了为止。

查询缓存默认是关闭。要想使用查询缓存，只需要在hibernate.cfg.xml中加入一条配置即可：
```xml
<property name="hibernate.cache.use_query_cache">true</property>
```
而且，我们在查询hql语句时，要想使用查询缓存，就需要在语句中设置这样一个方法：`setCacheable(true)`。关于这个的demo我就不进行演示了，大家可以自己慢慢试着玩一下。

但需要注意的是，我们在开启查询缓存的时候，也应该开启二级缓存。因为如果不使用二级缓存，也有可能出现N+1的问题。

这是因为查询缓存缓存的仅仅是对象的ID，所以首先会通过一条SQL将对象的ID都查询出来，但是当我们后面要得到每个对象的信息的时候，此时又会发送SQL语句，所以如果我们使用查询缓存，一定也要开启二级缓存。

#总结
这些就是自己今晚上研究的关于Hibernate缓存的一些问题，其出发点也是为了自己能够对Hibernate缓存的知识有一定的总结。当然了，下一步还需要深入到缓存是如何实现的这个深度中。

另外PS一句，最近打球打的很累，都感觉自己打的有点乏力了。休息几天再去玩。
