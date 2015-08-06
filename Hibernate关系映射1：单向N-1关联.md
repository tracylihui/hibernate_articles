title: Hibernate关系映射1：单向N-1关联
date: 2015-07-07 14:55:02
categories: Hibernate
tags: [Hibernate,关系映射]
---
我最近在复习一下关于Hibernate关系映射的知识，看了书本的介绍以及视频。这几篇博客都是对学到知识的一点总结。当然，这些这是最基本的能够实现关联关系的配置，在实际的使用中，还有很多参数需要根据情况来设置。但也算是对以后开发过程中遇到遗忘的地方可以进行查阅。

在本文中使用的Demo也都已经上传到github中，里边有详细的运行说明。
[Hibernate关系映射1：单向N-1关联](http://tracylihui.github.io/2015/07/07/Hibernate%E5%85%B3%E7%B3%BB%E6%98%A0%E5%B0%841%EF%BC%9A%E5%8D%95%E5%90%91N-1%E5%85%B3%E8%81%94/)
[Hibernate关系映射2：双向1-N关联](http://tracylihui.github.io/2015/07/07/Hibernate%E5%85%B3%E7%B3%BB%E6%98%A0%E5%B0%842%EF%BC%9A%E5%8F%8C%E5%90%911-N%E5%85%B3%E8%81%94/)
[Hibernate关系映射3：双向1-1关联](http://tracylihui.github.io/2015/07/07/Hibernate%E5%85%B3%E7%B3%BB%E6%98%A0%E5%B0%843%EF%BC%9A%E5%8F%8C%E5%90%911-1%E5%85%B3%E8%81%94/)
[Hibernate关系映射4：N-N关联](http://tracylihui.github.io/2015/07/08/Hibernate%E5%85%B3%E7%B3%BB%E6%98%A0%E5%B0%844%EF%BC%9AN-N%E5%85%B3%E8%81%94/)

Github地址：[HibernateRelationMapping](https://github.com/tracylihui/HibernateRelationMapping)
<!--more-->
#单向N-1关联
---
单向N-1关系，比如多个人对应一个地址，只需从人实体端可以找到对应的地址实体，无须关系某个地址的全部住户。

单向 n-1 关联只需从 n 的一端可以访问 1 的一端。
##域模型
从 Order 到 Customer 的多对一单向关联需要在Order 类中定义一个 Customer 属性, 而在 Customer 类中无需定义存放 Order 对象的集合属性
![图1](http://7xk5ao.com1.z0.glb.clouddn.com/mysql1.jpg)
##关系数据模型
ORDERS 表中的 CUSTOMER_ID 参照 CUSTOMER 表的主键
![图2](http://7xk5ao.com1.z0.glb.clouddn.com/mysql2.jpg)
#Demo
Hibernate 使用 <many-to-one> 元素来映射多对一关联关系
`<many-to-one name="customer" class="Customer" column="CUSTOMER_ID" cascade="all" />`

- name: 设定待映射的持久化类的属性的名字
- column: 设定和持久化类的属性对应的表的外键
- class：设定待映射的持久化类的属性的类型
- cascade：意味着系统将先自动级联插入主表记录，也就是说先持久化Customer对象，再持久化Person对象。开发时不建议使用该属性，建议使用手工的方式。

下面是实体对象，分为Customer（顾客）和Order（订单）,其中订单和顾客是N-1关系。
```java
public class Customer {
	private Integer customerId;
	private String customerName;
	//省去get和set方法
}
public class Order {
	private Integer orderId;
	private String orderName;
	private Customer customer;
        //省去get和set方法
}
```
Customer.hbm.xml
```xml
<hibernate-mapping>
    <class name="com.lihui.hibernate.single_n_1.Customer" table="CUSTOMERS">
        <id name="customerId" type="java.lang.Integer">
            <column name="CUSTOMER_ID" />
            <generator class="native" />
        </id>
        <property name="customerName" type="java.lang.String">
            <column name="CUSTOMER_NAME" />
        </property>
    </class>
</hibernate-mapping>
```
Order.hbm.xml
```xml
<hibernate-mapping package="com.lihui.hibernate.single_n_1">
	<class name="Order"
		table="ORDERS">
		<id name="orderId" type="java.lang.Integer">
			<column name="ORDER_ID" />
			<generator class="native" />
		</id>
		<property name="orderName" type="java.lang.String">
			<column name="ORDER_NAME" />
		</property>
		<many-to-one name="customer" class="Customer" column="CUSTOMER_ID" cascade="all" />
	</class>
</hibernate-mapping>
```
Junit Test
由于在Order.hbm.xml中配置了`cascade="all"`，所以只需要保存order对象即可，会首先自动保存级联的Customer对象。
```java
@Test
	public void testSave() {
		Customer customer = new Customer();
		customer.setCustomerName("a");

		Order order1 = new Order();
		order1.setOrderName("A");
		order1.setCustomer(customer);
		Order order2 = new Order();
		order2.setOrderName("B");
		order2.setCustomer(customer);

		session.save(order1);
		session.save(order2);
	}
```
