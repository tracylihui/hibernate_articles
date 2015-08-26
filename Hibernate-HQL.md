#导读
---
HQL(Hibernate Query Language) 是面向对象的查询语言, 它和 SQL 查询语言有些相似. 在 Hibernate 提供的各种检索方式中, HQL 是使用最广的一种检索方式. 它有如下功能:
1. 在查询语句中设定各种查询条件；
1. 支持投影查询, 即仅检索出对象的部分属性；
1. 支持分页查询；
1. 支持连接查询；
1. 支持分组查询, 允许使用 HAVING 和 GROUP BY 关键字；
1. 提供内置聚集函数, 如 sum(), min() 和 max()；
1. 支持子查询；
1. 支持动态绑定参数；
1. 能够调用 用户定义的 SQL 函数或标准的 SQL 函数。

<!--more-->
HQL 查询包括以下步骤:
1. 获取Hibernate Session对象。
2. 编写HQL语句
3. 以HQL语句作为参数，调用Session的createQuery方法创建查询对象。
3. 如果HQL语句包含参数，则调用Query的setXxx方法为参数赋值。
3. 调用Query对象的list()或uniqueResult()方法返回查询结果列表（持久化实体集）

Qurey 接口支持方法链编程风格, 它的 setXxx() 方法返回自身实例, 而不是 void 类型，因此可以写类似于`.setXxx().setXxx().setXxx()...`样式的语句。
#HQL vs SQL
---
HQL 查询语句是面向对象的, Hibernate 负责解析 HQL 查询语句, 然后根据对象-关系映射文件中的映射信息, 把 HQL 查询语句翻译成相应的 SQL 语句。HQL 查询语句中的主体是域模型中的类及类的属性。

SQL 查询语句是与关系数据库绑定在一起的。SQL 查询语句中的主体是数据库表及表的字段。
#HQL实用技术
---
##实体查询
最简单实体查询例子：
```java
String hql = "from User";
Query query = session.createQuery(hql);
List<User> list = query.list();
```
上面的HQL语句将取出User的所有对应记录为：`select user0_.U_ID as U_ID1_0_,user0_.U_NAME as U_NAME2_0_,user0_.U_AGE as U_AGE3_0_ from USERS user0_`

在HQL语句中，本身大小写无关，但是其中出现的类名和属性名必须注意大小写区分。同时，在Hibernate中，查询的目标实体存在继承关系的判定，如果`from User`将返回所有User以及User子类的记录。假设系统中存在User的两个子类：SysAdmin和SysOperator，那么该hql语句返回的记录将包含这两个子类的所有数据，即使SysAdmin和SysOperator分别对应了不同的库表。

Where子句：
如果我们想取出名为“Erica”的用户记录，可以通过Where子句加以限定(其中AS可以省略)：
```
FROM User AS user WHERE user.name='Erica'
```
where子句中，我们可以通过比较操作符指定甄选条件，如：
=, <>, <, >, <=, >=, between, not between, in ,not in, is, like等。同时，在where子句中可以使用算术表达式。
几个简单实例：
```
FROM User user WHERE user.age<20
FROM User user WHERE user.name IS null
FROM User user WHERE user.name LIKE 'Er%'
FROM User user WHERE (user.age % 2 = 1)
FROM User user WHERE (user.age<20) AND (user.name LIKE '%Er')
```
##属性查询
有时，我们需要的数据可能仅仅是实体对象的某个属性（库表记录中的某个字段信息）。通过HQL可以简单的做到这一点。
```java
String hql = "SELECT user.name FROM User user";
List list = session.createQuery(hql).list();
Iterator it = list.iterator();
while(it.hasNext()){
	System.out.println(it.next());
}
```
上例中，我们指定了只需要获取User的name属性。此时返回的list数据结构中，每个条目都是一个String类型的name数据。
我们也可以通过一条HQL获取多个属性：
```java
public void test() {
		String hql = "SELECT user.name,user.age FROM User user";
		List list = session.createQuery(hql).list();
		Iterator it = list.iterator();
		while(it.hasNext()){
			Object[] results = (Object[]) it.next();
			System.out.println(results[0]+","+results[1]);
		}
	}
```
而此时，返回的list数据结构中，每个条目都是一个对象数组(Object[])，其中依次包含了我们所获取的属性数据。

除此之外，我们也可以通过在HQL中动态的构造对象实例的方法对这些平面化的数据进行封装。
```java
String hql = "SELECT new User(user.uName,user.uAge) FROM User user";
		List list = session.createQuery(hql).list();
		Iterator it = list.iterator();
		while(it.hasNext()){
			User user = (User) it.next();
			System.out.println(user);
		}
```
通过在HQL中动态的构造对象实例，我们实现了对查询结果的对象化封装。此时在查询结果中的User对象仅仅是一个普通的Java对象，仅用于对查询结果的封装，除了在构造时赋予的属性值外，其他属性均为未赋值状态。同时，在实体类中，要提供包含构造属性的构造方法，并且顺序要相同。

与此同时，我们也可以在HQL的select子句中使用统计函数或者利用DSITINCT关键字，剔除重复记录。
```
SELECT COUNT(*),MIN(user.age) FROM User user
SELECT DISTINCT user.name FROM User user
```
##实体更新与删除
```java
String hql = "UPDATE User SET age = 18 WHERE id = 1";
		int result = session.createQuery(hql).executeUpdate();
```
上述代码利用HQL语句实现了更新操作。对于单个对象的更新也许代码量并没有减少太多，但如果对于批量更新操作，其便捷性以及性能的提高就相当可观。
例如，以下代码将所有用户的年龄属性更改为10
```
UPDATE User SET age = 18
```
HQL的delete子句使用同样很简单，例如以下代码删除了所有年龄大于18的用户记录：
```
DELETE User WHERE age > 18
```
不过，需要注意的是，在HQL delete/update子句的时候，必须特别注意它们对缓存策略的影响，极有可能导致缓存同步上的障碍。
##分组与排序
###Order by子句
举例说明：
```
FROM User user ORDER BY user.name
```
默认情况下是按照升序排序，当然我们可以指定排序策略:
```
FROM User user ORDER BY user.name DESC
```
order by子句可以指定多个排序条件：
```
FROM User user ORDER BY user.name, user.age DESC
```
###Group by子句
通过Group by可进行分组统计，如果下例中，我们通过Group by子句实现了同龄用户的统计：
```
SELECT COUNT(user),user.age FROM User user GROUP BY user.age
```
通过该语句，我们获得了一系列的统计数据。对于Group by子句获得的结果集而言，我们可以通过Having子句进行筛选。例如，在上例中，我们对同龄用户进行了统计，获得了每个年龄层次中的用户数量，假设我们只对超过10人的年龄组感兴趣，可用以下语句实现：
```
SELECT COUNT(user),user.age FROM User user GROUP BY user.age HAVING COUNT(user) > 10
```
##参数绑定
###SQL注入
在解释参数绑定的使用时，我们先来解释一下什么是SQL注入。

SQL Injection是常见的系统攻击手短，这种攻击方式的目标是针对由SQL字符串拼接造成的漏洞。如，为了实现用户登录功能，我们编写了以下代码：
```
FROM User user WHERE user.name='"+username+"' AND user.password='"+password+"'
```
从逻辑上讲，该HQL并没有错误，我们根据用户名和密码从数据库中读取相应的记录，如果找到记录，则认为用户身份合法。

假设这里的变量username和password是来自于网页上输入框的数据。现在我们来做个尝试，在登录网页上输入用户名："'Erica' or 'x'='x'"，密码随意，也可以登录成功。

此时的HQL语句为：
```
FROM User user WHERE user.name='Erica' OR 'x'='x' AND user.password='fasfas'
```
此时，用户名中的` OR 'x'='x'`被添加到了HQL并作为子句执行，where逻辑为真，而密码是否正确就无关紧要。

这就是SQL Injection攻击的基本原理，而字符串拼接而成的HQL是安全漏洞的源头。参数的动态绑定机制可以妥善处理好以上问题。

Hibernate提供顺序占位符以及引用占位符，将分别举例说明：

顺序占位符：
```java
String hql = "from User user WHERE user.name = ? AND user.age = ?";
		List<User> list = session.createQuery(hql).setString(0, "Erica")
				.setInteger(1, 10).list();
```
引用占位符：
```java
String hql = "from User user WHERE user.uName = :name AND user.uAge = :age";
		List<User> list = session.createQuery(hql).setString("name", "Erica")
				.setInteger("age", 10).list();
```
我们甚至还可以用一个JavaBean来封装查询参数。

参数绑定机制可以使得查询语法与具体参数数值相互独立。这样，对于参数不同，查询语法相同的查询操作，数据库即可实施性能优化策略。同时，参数绑定机制也杜绝了参数值对查询语法本身的影响，这也就是避免了SQL Injection的可能。
##引用查询
我们可能遇到过如下编码规范：“代码中不允许出现SQL语句”。

SQL语句混杂在代码之间将破坏代码的可读性，并似的系统的可维护性降低。为了避免这样的情况，我们通常采取将SQL配置化的方式，也就是说，将SQL保存在配置文件中。Hibernate提供了HQL可配置化的内置支持。

我们可以在实体映射文件中，通过query节点定义查询语句（与class节点同级）：
```xml
<query name="queryTest"><![CDATA[FROM User user where user.uAge < 20]]></query>
```
需要注意的是，我们是将HQL语句写入到了xml文件中，所以可能会造成冲突。例如HQL语句的的“<”（小于）与xml的语法有冲突。所以我们会用CDATA将其包裹。

之后我们可以通过session的getNamedQuery方法从配置文件中调用对应的HQL，如：
```java
Query query = session.getNamedQuery("queryTest");
		List<User> list = query.list();
		for(User user : list){
			System.out.println(user);
		}
```
##关联查询
关于这部分的内容，参考了很多书上的资料，但都感觉讲的不够清晰，也就是说没有结合到实际的情况中去。下面将按照一个视频教程上的顺序来介绍关联查询。

关于这部分的知识点，是鉴于已经对关联连接查询有所了解的基础上，比如懂得什么是左外连接、内连接等。
下面就开始总结：
###HQL迫切左外连接
`LEFT JOIN FETCH`关键字表示使用迫切左外连接策略。
首先看一下例子中的实体类，这是一个双向1-N的映射（关于1-N的映射在之前的博客中有介绍[Hibernate关系映射2：双向1-N关联](http://tracylihui.github.io/2015/07/07/Hibernate%E5%85%B3%E7%B3%BB%E6%98%A0%E5%B0%842%EF%BC%9A%E5%8F%8C%E5%90%911-N%E5%85%B3%E8%81%94/)）.

实体类：
```java
public class Department {

	private Integer id;
	private String name;
	private Set<Employee> emps = new HashSet<>();
  //省却get和set方法
}
public class Employee {

	private Integer id;
	private String name;
	private float salary;
	private String email;

	private Department dept;
  //省去get和set方法
}
```
####迫切左外连接
```java
String hql = "SELECT DISTINCT d FROM Department d LEFT JOIN FETCH d.emps";
Query query = session.createQuery(hql);
List<Department> depts = query.list();
System.out.println(depts.size());
for (Department dept : depts) {
	System.out.println(dept.getName() + "-" + dept.getEmps().size());
}
```
上面的例子中是想得到所有的部门以及其中的员工。我们通过DISTINCT进行去重。
注意的点：
- list()方法中返回的集合中存放实体对象的引用。每个Department对象关联的Employee集合都将被初始化，存放所有关联的Employee的实体对象
- 查询结果中可能会包含重复元素，可以通过DISTINCT关键字去重，同样由于list中存放的实体对象的引用，所以可以通过HashSet来过滤重复对象。例如：

```java
List<Department> depts = query.list();
depts = new ArrayList<>(new LinkedHashSet(depts));
```
####左外连接
```java
String hql = "FROM Department d LEFT JOIN d.emps";
Query query = session.createQuery(hql);
List<Object[]> result = query.list();
System.out.println(result);
for (Object[] objs : result) {
	System.out.println(Arrays.asList(objs));
}
```
注意的是：通过左外连接返回的list是一个包含Department和与之连接的Employee的object数组。所以我们还需要对数组进行处理，并且有重复。鉴于此，我们可以通过DISTINCT进行处理。
```java
String hql = "SELECT DISTINCT d FROM Department d LEFT JOIN d.emps";
		Query query = session.createQuery(hql);

		List<Department> depts = query.list();
		System.out.println(depts.size());

		for (Department dept : depts) {
			System.out.println(dept.getName() + ", " + dept.getEmps().size());
		}
```
注意：
- list方法返回的集合中存放的是对象数组类型
- 根据配置文件来决定Employee集合的检索策略，比如是fetch、lazy啊或者怎样，还不是像迫切左外连接那样。

我们真正进行的过程中，一般都会使用迫切左外连接。因为迫切左外连接只发送了一条SQL语句将所有信息都查出来，而左外连接就算不使用集合的对象，也会进行查询，而当真正使用集合的时候，又会再去查询。所以性能上迫切左外连接要好。
##子查询
子查询可以在HQL中利用另外一条HQL的查询结果。
例如：
```
FROM User user WHERE (SELECT COUNT(*) FROM user.address) > 1
```
HQL中，子查询必须出现在where子句中，且必须以一对圆括号包围。
