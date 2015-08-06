title: Hibernate：检索策略的学习1
date: 2015-07-10 13:01:54
categories: Hibernate
tags: [Hibernate,检索策略]
---
#概述
---
检索数据，也就是查询数据是在一个系统中必不可少的一个功能。检索数据时的2个问题：
1. 不浪费内存：例如，Customer和Order是双向1-N的关系。当 Hibernate 从数据库中加载 Customer 对象时， 如果同时加载所有关联的 Order 对象， 而程序实际上仅仅需要访问 Customer 对象， 那么这些关联的 Order 对象就白白浪费了许多内存。
2. 更高的查询效率：发送尽可能少的 SQL 语句。

由于篇幅原因，将内容分为了两部分：

[Hibernate：检索策略的学习1](http://tracylihui.github.io/2015/07/10/Hibernate%EF%BC%9A%E6%A3%80%E7%B4%A2%E7%AD%96%E7%95%A5%E7%9A%84%E5%AD%A6%E4%B9%A01)
[Hibernate：检索策略的学习2](http://tracylihui.github.io/2015/07/10/Hibernate%EF%BC%9A%E6%A3%80%E7%B4%A2%E7%AD%96%E7%95%A5%E7%9A%84%E5%AD%A6%E4%B9%A02/)

第一部分讲解了类级别的检索策略以及1-N和N-N的检索策略，在第二部分中将学习N-1和1-1的检索策略，并对检索策略进行总结。
<!--more-->
#类级别的检索策略
---
##知识点
类级别可选的检索策略包括立即检索和延迟检索， 默认为延迟检索。
- 立即检索: 立即加载检索方法指定的对象；
- 延迟检索: 延迟加载检索方法指定的对象。在使用具体的属性时，再进行加载。

**类级别的检索策略可以通过 <class> 元素的 lazy 属性进行设置。**

如果程序加载一个对象的目的是为了访问它的属性，可以采取立即检索；如果程序加载一个持久化对象的目的是仅仅为了获得它的引用，可以采用延迟检索，但**需要注意懒加载异常**（LazyInitializationException：简单理解该异常就是Hibernate在使用延迟加载时，并没有将数据实际查询出来，而只是得到了一个代理对象，当使用属性的时候才会去查询，而如果这个时候session关闭了，则会报该异常）！

下面通过一个例子来进行讲解：
##Demo
在该Demo中，我们只需要使用一个Customer的对象即可，其中包含了id，name等属性。
###延迟检索
首先我们来测试一下<class>元素的lazy属性为true的情况，也就是默认情况（不设置）。
```java
public class HibernateTest {

	private SessionFactory sessionFactory;
	private Session session;
	private Transaction transaction;
	@Before
	public void init() {
		Configuration configuration = new Configuration().configure();
		ServiceRegistry serviceRegistry = new ServiceRegistryBuilder()
				.applySettings(configuration.getProperties())
				.buildServiceRegistry();
		sessionFactory = configuration.buildSessionFactory(serviceRegistry);

		session = sessionFactory.openSession();
		transaction = session.beginTransaction();
	}
	@After
	public void destroy() {
		transaction.commit();
		session.close();
		sessionFactory.close();
	}
	@Test
	public void testClassLevelStrategy() {
		Customer customer = (Customer) session.load(Customer.class, 1);
		System.out.println(customer.getClass());
		System.out.println(customer.getCustomerId());
		System.out.println(customer.getCustomerName());
	}
}
```
看一下上面的代码，该代码是利用Junit进行测试（关于Junit的知识在这不多说，并不是重点）。其中init方法是对SessionFactory、Session等进行初始化，destroy方法是进行关闭。
当我们运行testClassLevelStrategy()方法时，会得到以下输出结果：
```
class com.atguigu.hibernate.strategy.Customer_$$_javassist_1
1
Hibernate:
    select
        customer0_.CUSTOMER_ID as CUSTOMER1_0_0_,
        customer0_.CUSTOMER_NAME as CUSTOMER2_0_0_
    from
        CUSTOMERS customer0_
    where
        customer0_.CUSTOMER_ID=?
AA
```
通过控制台的输出结果，我们可以清楚的看到，当我们执行`session.load(Customer.class, 1)`方法时，Hibernate并没有发送SQL语句，而只是返回了一个对象，通过调用getClass()方法，可以看到该对象`class com.atguigu.hibernate.strategy.Customer_$$_javassist_1`是一个代理对象，并且当调用`customer.getCustomerId()`获取ID的时候，也没有发送SQL语句；当我们这个再调用`customer.getCustomerName()`方法来得到name的时候，这个时候才发送了SQL语句进行真正的查询，并且WHERE条件中带上的就是ID。

如果我们在`System.out.println(customer.getCustomerName());`语句前插入`session.close()`将Session关闭，就能看到之前上文中提到的懒加载异常。
###立即检索
为了让Customer类可以立即检索，我们要修改Customer.hbm.xml文件，在<class>标签中加入lazy="false"。
```xml
<hibernate-mapping package="com.atguigu.hibernate.strategy">

	<class name="Customer" table="CUSTOMERS" lazy="false">

		<id name="customerId" type="java.lang.Integer">
			<column name="CUSTOMER_ID" />
			<generator class="native" />
		</id>
    ...
```
这个时候，我们再运行之前的单元测试代码，控制台会得到以下输出结果：
```
Hibernate:
    select
        customer0_.CUSTOMER_ID as CUSTOMER1_0_0_,
        customer0_.CUSTOMER_NAME as CUSTOMER2_0_0_
    from
        CUSTOMERS customer0_
    where
        customer0_.CUSTOMER_ID=?
class com.atguigu.hibernate.strategy.Customer
1
AA
```
我们可以看到，当调用load方法时，会发送SQL语句，并且得到的不再是代理对象。这个时候就算我们在输出name属性之前将session关闭，也不会报错。
##小结
**上文中对Hibernate的类级别的检索策略进行了总结，当然这些是建立在有一定基础的前提下。**
需要注意的是：
-  无论<class>元素的lazy 属性是true 还是false，Session 的get() 方法及Query 的list() 方法在类级别总是使用立即检索策略。
-  若<class> 元素的 lazy 属性为 true 或取默认值, Session 的 load() 方法不会执行查询数据表的 SELECT 语句，仅返回代理类对象的实例，该代理类实例有如下特征：
  - 由 Hibernate 在运行时采用 CGLIB 工具动态生成；
  - Hibernate 创建代理类实例时，仅初始化其 OID 属性；
  - 在应用程序第一次访问代理类实例的非 OID 属性时, Hibernate 会初始化代理类实例。

#1-N和N-N的检索策略
---
##知识点
我在之前的博客中对1-N和N-N有过学习，所以我假设读者已经了解了Hibernate中关于1-N和N-N的映射。我们建立了Customer与Order的1-N关联关系，表示一个顾客可以有多个订单。

我们在映射文件中，用<set>元素来配置1-N关联以及N-N关联关系。<set>元素有lazy、fetch和batch-size属性。
- lazy: 主要决定orders 集合被初始化的时机。即到底是在加载Customer 对象时就被初始化, 还是在程序访问 orders 集合时被初始化。
- fetch: 取值为 “select” 或 “subselect” 时, 决定初始化 orders 的查询语句的形式；若取值为”join”, 则决定 orders 集合被初始化的时机
- 若把 fetch 设置为 “join”， lazy 属性将被忽略
- <set> 元素的 batch-size 属性：用来为延迟检索策略或立即检索策略设定批量检索的数量. 批量检索能减少 SELECT 语句的数目， 提高延迟检索或立即检索的运行性能。

|lazy属性<br>(默认值true)|fetch属性<br>(默认值select)|检索策略|
|-------------|:-------------:| -----|
|true |未设置<br>(取默认值select)| 采用延迟检索策略，这是默认的检索策略，也是优先考虑使用的检索策略|
|false|未设置<br>(取默认值select)| 采用立即索策略，当使用Hibernate二级缓存时可以考虑使用立即检索|
|extra|未设置<br>(取默认值select)| 采用加强延迟检索策略，它尽可能的延迟orders集合被初始化的时机|
|true,extra<br> or extra|未设置<br>(取默认值select)| lazy属性决定采用的检索策略，即决定初始化orders集合的时机。fetch属性为select，意味<br>着通过select语句来初始化orders的集合，形式为SELECT * FROM orders WHERE customer<br>_id IN (1,2,3,4)|
|true,extra<br> or extra|subselect| lazy属性决定采用的检索策略，即决定初始化orders集合的时机。fetch属性为subselect，意味<br>着通过subselect语句来初始化orders的集合，形式为SELECT * FROM orders WHERE<br> customer_id IN (SELECT id FROM customers)|
|true|join| 采采用迫切左外连接策略|
##Lazy
我们现在开始研究一下关于<set>元素的lazy属性。
###Demo
首先我们看一下延迟检索，也就是<set>属性的lazy为true或者不设置的情况下：
```java
@Test
public void testOne2ManyLevelStrategy() {
		Customer customer = (Customer) session.get(Customer.class, 1);
		System.out.println(customer.getCustomerName());
    System.out.println(customer.getOrders().getClass());
		System.out.println(customer.getOrders().size());
}
```
下面是控制的输出结果
```
Hibernate:
    select
        customer0_.CUSTOMER_ID as CUSTOMER1_0_0_,
        customer0_.CUSTOMER_NAME as CUSTOMER2_0_0_
    from
        CUSTOMERS customer0_
    where
        customer0_.CUSTOMER_ID=?
AA
class org.hibernate.collection.internal.PersistentSet
Hibernate:
    select
        orders0_.CUSTOMER_ID as CUSTOMER3_0_1_,
        orders0_.ORDER_ID as ORDER_ID1_1_1_,
        orders0_.ORDER_ID as ORDER_ID1_1_0_,
        orders0_.ORDER_NAME as ORDER_NA2_1_0_,
        orders0_.CUSTOMER_ID as CUSTOMER3_1_0_
    from
        ORDERS orders0_
    where
        orders0_.CUSTOMER_ID=?
3
```
从结果中可以明显的看出，Hibernate使用了延迟检索。其中的orders并没有初始化，而是返回了一个集合代理对象。当我们通过customer.getOrders().size()这段代码真正要使用orders集合的时候，才发送SQL语句进行查询。

在延迟检索（lazy属性值为true）集合属性时，Hibernate在以下情况下初始化集合代理类实例：
- 应用程序第一次访问集合属性: iterator(), size(), isEmpty(), contains() 等方法
- 通过 Hibernate.initialize() 静态方法显式初始化

下面我们将<set>的lazy属性修改为false，如`<set name="orders" table="ORDERS" inverse="true" lazy="false">`。

修改完之后再执行测试代码，输出结果也就很明显了，在调用load()方法时，会先执行SQL语句取出Customer以及相关联的orders。

最后，提一下lazy的另一个取值extra。
该取值与true类似，主要区别是增强延迟检索策略能够进一步延迟Customer对象的orders集合代理实例的初始化时机。

首先我们将<set>元素中的lazy设为extra。我们同样的执行上文中的单元测试代码， 得到以下结果：
```
Hibernate:
    select
        customer0_.CUSTOMER_ID as CUSTOMER1_0_0_,
        customer0_.CUSTOMER_NAME as CUSTOMER2_0_0_
    from
        CUSTOMERS customer0_
    where
        customer0_.CUSTOMER_ID=?
AA
class org.hibernate.collection.internal.PersistentSet
Hibernate:
    select
        count(ORDER_ID)
    from
        ORDERS
    where
        CUSTOMER_ID =?
3
```
我们观察第二个SQL语句。我们发现他并没有对orders进行初始化，而是通过使用一个count()函数。extra取值为增强的延迟检索，该取值会尽可能的延迟集合初始化的时机。

例如：当我们将lazy设置为true（延迟检索），而我们调用order.size()方法的时候，这个时候就会通过SQL将orders集合初始化。但现在我们用extra这个属性，发现我们调用orders的size方法，并没有初始化，而是通过了一个count函数。

**所以我们得到结论：**

增强延迟检索策略能进一步延迟 Customer 对象的 orders 集合代理实例的初始化时机：
- 当程序第一次访问 order 属性的 size(), contains() 和 isEmpty() 方法时, Hibernate 不会初始化 orders 集合类的实例, 仅通过特定的 select 语句查询必要的信息, 不会检索所有的 Order 对象。
- 当程序第一次访问 orders 属性的 iterator() 方法时, 会导致 orders 集合代理类实例的初始化。（这个是没办法的= =）

但其实我们在实际的开发过程中，当我们要用到size或者contains等方法的时候，基本上代表我们就要用到集合部分的属性。如果我们选用extra的话，反倒会多发送SQL语句。

关于extra的其他点大家可以自己进行一些测试，比较简单方便。
##BatchSize
<set> 元素有一个batch-size 属性，用来为延迟检索策略或立即检索策略设定批量检索的数量。批量检索能减少 SELECT 语句的数目，提高延迟检索或立即检索的运行性能。
###Demo
首先说一下这个Demo中用到的数据的数据，Customers表中共有4条数据。而我们的lazy属性设为true，即采用延迟加载策略。
我们看一下下面的代码：
```java
@Test
public void testSetBatchSize() {
	List<Customer> customers = session.createQuery("FROM Customer").list();
	System.out.println(customers.size());
	for (Customer customer : customers) {
		if (customer.getOrders() != null)
			System.out.println(customer.getOrders().size());
	}
}
```
首先，在该代码的第三行我们获取了所有的Customer，得到list集合（集合中存的是Customer对象），然后遍历该list集合；每次循环，我们都通过customer.getOrders方法来实例化orders集合。

篇幅原因，我在这不贴控制台的输出了。但可以想到的是，会一共发出5条SQL。第一条是获取customer集合，因为集合有4个customer对象，所以会再发出4条SQL来分别初始化。

好了，现在我们修改<set>元素，在里边加入`batch-size="4"`，所以现在的<set>元素的代码为`<set name="orders" table="ORDERS" inverse="true" lazy="true" batch-size="4">`。

我们重新运行单元测试代码，得到结果：
```
Hibernate:
    select
        customer0_.CUSTOMER_ID as CUSTOMER1_0_,
        customer0_.CUSTOMER_NAME as CUSTOMER2_0_
    from
        CUSTOMERS customer0_
4
Hibernate:
    select
        orders0_.CUSTOMER_ID as CUSTOMER3_0_1_,
        orders0_.ORDER_ID as ORDER_ID1_1_1_,
        orders0_.ORDER_ID as ORDER_ID1_1_0_,
        orders0_.ORDER_NAME as ORDER_NA2_1_0_,
        orders0_.CUSTOMER_ID as CUSTOMER3_1_0_
    from
        ORDERS orders0_
    where
        orders0_.CUSTOMER_ID in (
            ?, ?, ?, ?
        )
3
3
3
0
```
我们看到这个时候只有2条SQL语句。第一条同样的还是得到customer，而第二条SQL语句直接全部初始化了4个orders集合。这就是batch-size的作用所在。我们可以想到的是，如果我们将batch-size设为2，则是3条SQL语句。**也就是我们上文中提到的批量检索。**
##Fetch
<set>元素的fetch属性是用于确定初始化orders 集合的方式。
1. 默认值为orders，也就是通过正常的方式来初始化set元素。例如我们在将batch-size例子的时候，我们并没有设置fetch属性（默认即为select），所以我们在初始化orders集合的时候，会发现在SQL语句中是通过`where orders0_.CUSTOMER_ID in (?, ?, ?, ?)`这种方式来进行初始化的。
2. 可以取值为subselect，我们看名字也能知道大概意思，就是通过子查询的方式来初始化所有的set集合。例如我们设置为subselect后，可看到SQL语句中包含`where orders0_.CUSTOMER_ID in (select customer0_.CUSTOMER_ID from CUSTOMERS customer0_)`。**子查询作为WHERE子句的in的条件出现的，子查询查询1的一端的ID，此时lazy属性是有效的，但batch-size属性失效。**
3. 若取值为join：
  + 在加载1的一端的对象的时候，使用迫切左外连接（可参考该博客进行学习[Hibernate:深入HQL学习](http://tracylihui.github.io/2015/07/08/Hibernate%EF%BC%9A%E6%B7%B1%E5%85%A5HQL%E5%AD%A6%E4%B9%A0/)）的方式检索N的一端的集合的属性；
  + 忽略lazy属性(理解了迫切左外连接，也就能知道为啥会忽略lazy属性了)；
  + HQL查询忽略`fetch="join"`的取值；
	+ **Query 的list() 方法会忽略映射文件中配置的迫切左外连接检索策略, 而依旧采用延迟加载策略。（这个点之前测试的时候老是忽略掉了）**

#总结
以上就是关于Hibernate检索策略的学习，但并不全。将在下一篇中，对N-1和1-1的检索策略进行学习，并做一个总结。
