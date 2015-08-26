我最近在复习一下关于Hibernate关系映射的知识，看了书本的介绍以及视频。这几篇博客都是对学到知识的一点总结。当然，这些这是最基本的能够实现关联关系的配置，在实际的使用中，还有很多参数需要根据情况来设置。但也算是对以后开发过程中遇到遗忘的地方可以进行查阅。

在本文中使用的Demo也都已经上传到github中，里边有详细的运行说明。
[Hibernate关系映射1：单向N-1关联](http://tracylihui.github.io/2015/07/07/Hibernate%E5%85%B3%E7%B3%BB%E6%98%A0%E5%B0%841%EF%BC%9A%E5%8D%95%E5%90%91N-1%E5%85%B3%E8%81%94/)
[Hibernate关系映射2：双向1-N关联](http://tracylihui.github.io/2015/07/07/Hibernate%E5%85%B3%E7%B3%BB%E6%98%A0%E5%B0%842%EF%BC%9A%E5%8F%8C%E5%90%911-N%E5%85%B3%E8%81%94/)
[Hibernate关系映射3：双向1-1关联](http://tracylihui.github.io/2015/07/07/Hibernate%E5%85%B3%E7%B3%BB%E6%98%A0%E5%B0%843%EF%BC%9A%E5%8F%8C%E5%90%911-1%E5%85%B3%E8%81%94/)
[Hibernate关系映射4：N-N关联](http://tracylihui.github.io/2015/07/08/Hibernate%E5%85%B3%E7%B3%BB%E6%98%A0%E5%B0%844%EF%BC%9AN-N%E5%85%B3%E8%81%94/)

Github地址：[HibernateRelationMapping](https://github.com/tracylihui/HibernateRelationMapping)
<!--more-->

# 单向N-N关联

---
N-N关联映射增加一张表才完成基本映射。

与1-N映射相似，必须为set集合元素添加key子元素，指定CATEGORIES_ITEMS表中参照CATEGORIES表的外键为CATEGORIY_ID。

与1-N不同的是，建立N-N关联时，集合中的元素使用many-to-many。关于配置文件的属性的介绍，将在代码实现部分介绍。

## 域模型

![图1](http://7xk5ao.com1.z0.glb.clouddn.com/nn1.png)

## 关系数据模型

![图2](http://7xk5ao.com1.z0.glb.clouddn.com/nn2.png)

## 代码实现

### 实体类

```java
public class Category {
	private Integer categoryId;
	private String catregoryName;

	private Set<Item> items = new HashSet<Item>();
  //省去get和set方法
}
public class Item {
	private Integer itemId;
	private String itemName;
	//省去get和set方法
}
```

### Category.hbm.xml

```xml
<hibernate-mapping package="com.lihui.hibernate.single_n_n">
    <class name="Category" table="CATEGORIES">
        <id name="categoryId" type="java.lang.Integer">
            <column name="CATEGORY_ID" />
            <generator class="native" />
        </id>
        <property name="catregoryName" type="java.lang.String">
            <column name="CATREGORY_NAME" />
        </property>
        <set name="items" table="CATEGORIES_ITEMS">
        	<key>
        		<column name="C_ID"></column>
        	</key>
        	<many-to-many class="Item" column="I_ID"></many-to-many>
        </set>
    </class>
</hibernate-mapping>
```
- table:指定中间表
- many-to-many:指定多对多的关联关系
- column:执行set集合中的持久化类在中间表的外键列的名称

### Item.hbm.xml

```xml
<hibernate-mapping package="com.lihui.hibernate.single_n_n">
    <class name="Item" table="ITEMS">
        <id name="itemId" type="java.lang.Integer">
            <column name="ITEM_ID" />
            <generator class="native" />
        </id>
        <property name="itemName" type="java.lang.String">
            <column name="ITEM_NAME" />
        </property>
    </class>
</hibernate-mapping>
```
### Junit

```java
@Test
	public void testSave() {
		Category c1 = new Category();
		c1.setCatregoryName("C-AA");

		Category c2 = new Category();
		c2.setCatregoryName("C-BB");

		Item i1 = new Item();
		i1.setItemName("I-AA");
		Item i2 = new Item();
		i2.setItemName("I-BB");

		c1.getItems().add(i1);
		c1.getItems().add(i2);
		c2.getItems().add(i1);
		c2.getItems().add(i2);

		session.save(c1);
		session.save(c2);
		session.save(i1);
		session.save(i2);

	}
```

# 双向N-N关联

---
双向N-N关联需要两端都使用set集合属性，两端都增加对集合属性的访问。

在双向N-N关联的两边都需指定连接表的表名及外键列的列名. 两个集合元素 set 的 table 元素的值必须指定，而且必须相同。set元素的两个子元素：key 和 many-to-many 都必须指定 column 属性，其中，key 和 many-to-many 分别指定本持久化类和关联类在连接表中的外键列名，因此两边的 key 与 many-to-many 的column属性交叉相同。也就是说，一边的set元素的key的 cloumn值为a,many-to-many 的 column 为b；则另一边的 set 元素的 key 的 column 值 b,many-to-many的 column 值为 a。  
![图3](http://7xk5ao.com1.z0.glb.clouddn.com/nn3.png)
对于双向 n-n 关联, <font color="#ff0000">必须把其中一端的 inverse 设置为 true, 否则两端都维护关联关系可能会造成主键冲突。</font>

## 域模型

![图4](http://7xk5ao.com1.z0.glb.clouddn.com/nn4.png)

## 关系数据模型

![图5](http://7xk5ao.com1.z0.glb.clouddn.com/nn5.png)

## 代码实现

### 实体类

与上文中实体类相似，但是在Item中增加了category的set集合。

```java
public class Category {
	private Integer categoryId;
	private String catregoryName;

	private Set<Item> items = new HashSet<Item>();
  //省去get和set方法
}
public class Item {
	private Integer itemId;
	private String itemName;
	//省去get和set方法
}
```

### Category.hbm.xml

与上文中相同，不再复述。

### Item.hbm.xml

```xml
<hibernate-mapping package="com.lihui.hibernate.single_n_n">
    <class name="Item" table="ITEMS">
        <id name="itemId" type="java.lang.Integer">
            <column name="ITEM_ID" />
            <generator class="native" />
        </id>
        <property name="itemName" type="java.lang.String">
            <column name="ITEM_NAME" />
        </property>
        <set name="categories" table="CATEGORIES_ITEMS" inverse="true">
        	<key>
        		<column name="I_ID"></column>
        	</key>
        	<many-to-many class="Category" column="C_ID"></many-to-many>
        </set>
    </class>
</hibernate-mapping>
```

### Notice

注意要在其中一端加入`inverse="true"`，否则会造成主键冲突。
