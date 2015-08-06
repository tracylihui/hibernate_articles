title: Hibernate关系映射3：双向1-1关联
date: 2015-07-07 17:32:38
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
#双向1-1关联
---
双向1-1关联需要修改两边的持久化类代码，让两个持久化类都增加引用关联实体的属性，并为该属性提供get和set方法。

双向1-1关联有三种映射模式：
- 基于主键
- 基于外键
- 使用连接表

在该文中，将主要介绍前两种模式。
##域模型
![图1](http://7xk5ao.com1.z0.glb.clouddn.com/tt1.png)
##关系数据模型
###按照外键映射
![图2](http://7xk5ao.com1.z0.glb.clouddn.com/tt2.png)
###按照主键映射
![图3](http://7xk5ao.com1.z0.glb.clouddn.com/tt3.png)
#基于外键
---
##介绍
对于基于外键的1-1关联，外键可以存放在任意一边。需要存放外键的一端，需要增加`<many-to-one.../>`元素，并且为`<many-to-one.../>`元素增加`unique="true"`属性来表示该实体实际上是1的一端。
`<many-to-one name="manager" class="Manager" column="MGR_ID" unique="true"></many-to-one>`

对于1-1的关联关系，两个实体原本处于平等状态，但当我们选择任意一个表来增加外键后（增加`<many-to-one.../>`元素的实体端），该表即变成从表，而另一个表则成为主表。

另一端需要使用`<one-to-one.../>`元素，该`<one-to-one.../>`元素需要使用name属性指定关联属性名。为了让系统不再为本表增加一列，而是使用外键关联，使用`property-ref`属性指定引用关联类的属性。
`<one-to-one name="department" class="Department" property-ref="manager"></one-to-one>`
##代码
实体类
```java
public class Department {
	private Integer deptId;
	private String deptName;
	private Manager manager;
	//省去get和set
}
public class Manager {
	private Integer mgrId;
	private String mgrName;
	private Department department;
	//省去get和set
}
```
Manager.hbm.xml
```xml
<hibernate-mapping package="com.lihui.hibernate.double_1_1.foreign">
    <class name="Manager" table="MANAGERS">
        <id name="mgrId" type="java.lang.Integer">
            <column name="MGR_ID" />
            <generator class="native" />
        </id>
        <property name="mgrName" type="java.lang.String">
            <column name="MGR_NAME" />
        </property>
        <!-- 映射1-1的关联关系，在对应的数据表中已经有外键，当前持久花类使用ont-to-one进行映射 -->
        <one-to-one name="department" class="Department" property-ref="manager"></one-to-one>
    </class>
</hibernate-mapping>
```
Department.hbm.xml
```xml
<hibernate-mapping package="com.lihui.hibernate.double_1_1.foreign">
    <class name="Department" table="DEPARTMENTS">
        <id name="deptId" type="java.lang.Integer">
            <column name="DEPT_ID" />
            <generator class="native" />
        </id>
        <property name="deptName" type="java.lang.String">
            <column name="DEPT_NAME" />
        </property>
        <many-to-one name="manager" class="Manager" column="MGR_ID" unique="true"></many-to-one>
    </class>
</hibernate-mapping>
```
测试
```java
@Test
	public void testGet(){
		Department dep = (Department) session.get(Department.class, 1);
		System.out.println(dep.getDeptName());
		Manager mgr =dep.getManager();
		System.out.println(mgr.getMgrName());
	}
	@Test
	public void testSave() {
		Department dep = new Department();
		dep.setDeptName("a");
		Manager mgr = new Manager();
		mgr.setMgrName("b");
		dep.setManager(mgr);
		mgr.setDepartment(dep);
		session.save(mgr);
		session.save(dep);
	}
```
##Notice
上面的映射策略可以互换，即让Manager存放外键，使用`<many-to-one.../>`元素映射关联属性；但Department端则必须使用`<one-to-one.../>`元素映射，万万不可两边都使用相同的元素映射关联属性。
#基于主键
---
##介绍
如果采用基于主键的映射策略，则一端的主键生成器需要使用foreign策略，表明将根据对方的主键来生成自己的主键，本实体不能拥有自己的主键声称策略。`<param>`子元素指定使用当前持久化类的哪个属性作为“对方”。
```xml
<generator class="foreign" >
  <param name="property">manager</param>
</generator>
```
当然，任意一端都可以采用foreign主键生成器策略，表明将根据对方主键来生成自己的主键。

采用foreign主键生成器策略的一端增加one-to-one元素映射相关属性，其ont-to-one属性还应增加`constrained=true`属性；另一端增加one-to-one元素映射关联属性。

constrained：指定为当前持久化类对应的数据库表的主键添加一个外键约束，引用被关联对象所对应的数据库主键。
##代码
实体类与上文中的实体类相同。
Manager.hbm.xml
```xml
<hibernate-mapping package="com.lihui.hibernate.double_1_1.primary">
    <class name="Manager" table="MANAGERS">
        <id name="mgrId" type="java.lang.Integer">
            <column name="MGR_ID" />
            <generator class="native" />
        </id>
        <property name="mgrName" type="java.lang.String">
            <column name="MGR_NAME" />
        </property>
        <one-to-one name="department" class="Department"></one-to-one>
    </class>
</hibernate-mapping>
```
Department.hbm.xml
```xml
<hibernate-mapping package="com.lihui.hibernate.double_1_1.primary">
    <class name="Department" table="DEPARTMENTS">
        <id name="deptId" type="java.lang.Integer">
            <column name="DEPT_ID" />
            <generator class="foreign" >
            	<param name="property">manager</param>
            </generator>
        </id>
        <property name="deptName" type="java.lang.String">
            <column name="DEPT_NAME" />
        </property>
        <one-to-one name="manager" class="Manager" constrained="true"></one-to-one>
    </class>
</hibernate-mapping>
```
