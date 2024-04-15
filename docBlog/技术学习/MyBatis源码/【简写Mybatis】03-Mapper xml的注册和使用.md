## 前言

**在学习MyBatis文章中，斗胆想将其讲明白；故有此文章，如有问题，不吝指教！**

**注意：**

> 学习源码一定一定不要太关注代码的编写，而是注意代码实现思想；
>
> 通过设问方式来体现代码中的思想；方法：**5W+1H**

**源代码：**[**https://gitee.com/xbhog/mybatis-xbhog**](https://gitee.com/xbhog/mybatis-xbhog)**；**[**https://github.com/xbhog/mybatis-xbhog**](https://github.com/xbhog/mybatis-xbhog)**；交个朋友，欢迎star。**

## 回顾&分析

上一局实现[【简写Mybatis】02-注册机的实现以及SqlSession处理](https://blog.csdn.net/weixin_43908900/article/details/136293075?spm=1001.2014.3001.5501);主要是为了完善Mapper的注册方式以及SqlSession规范的流程；

上一局的测试类如下；在使用上还是比我们平常熟悉的Mybatis要差好多，比如没有Mapper配置文件、Mapper映射文件、没有针对数据库操作等。

```java
/**
 * Unit test for simple App.
 */
public class AppTest extends TestCase {
    /**
     * Rigourous Test :-)
     */
    public void testApp() {
        MapperRegistry mapperRegistry = new MapperRegistry();
        mapperRegistry.addMapper("com.xbhog");
        DefaultSqlSessionFactory sqlSessionFactory = new DefaultSqlSessionFactory(mapperRegistry);
        SqlSession sqlSession = sqlSessionFactory.openSession();
        IUserDao user = sqlSession.getMapper(IUserDao.class);
        String userName = user.getUserName("xbhog");
        System.out.println("输出的信息："+userName);
    }
}
```

## 目的

1. XML的解析和读取
2. 封装XML数据
3. 封装配置类
4. 封装MapperRegistry和SqlSessionFactory，设置统一入口
5. 封装Mapper文件中方法的执行操作

## 实现

### XML的解析和读取

引入相关的Maven依赖：

```xml
<dependency>
    <groupId>org.dom4j</groupId>
    <artifactId>dom4j</artifactId>
    <version>2.1.3</version>
</dependency>
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-all</artifactId>
    <version>5.5.0</version>
</dependency>
```

解析的相关代码如下：已有注释

```java
// xml文件内容转换为字符串流
Reader reader = ResourceUtil.getUtf8Reader("mybatis-config-datasource.xml");

// 从字符串流中读取并创建XML Document对象
SAXReader saxReader = new SAXReader();
Document document = saxReader.read(new InputSource(reader));

// 获取XML文档的根元素
Element rootElement = document.getRootElement();

// 获取根元素下的“mappers”子元素
Element mappersElement = rootElement.element("mappers");

// 遍历所有“mapper”子元素
List<Element> mapperElements = mappersElement.elements("mapper");
for (Element mapperElement : mapperElements) {
    // 获取当前“mapper”元素的“resource”属性值
    String mapperResource = mapperElement.attributeValue("resource");
    System.out.println("正在查看资源：" + mapperResource);

    // 依据“resource”属性加载对应的XML文件内容为字符串流
    Reader mapperReader = ResourceUtil.getUtf8Reader(mapperResource);

    // 创建新的SAXReader实例以读取mapper文件中的XML内容
    SAXReader saxReaderForMapper = new SAXReader();

    // 从mapper的字符串流中创建新的Document对象
    Document mapperDocument = saxReaderForMapper.read(new InputSource(mapperReader));

    // 获取mapper文件的根元素
    Element mapperRootElement = mapperDocument.getRootElement();

    // 遍历mapper文件中所有的“select”元素
    List<Element> selectNodes = mapperRootElement.elements("select");
    for (Element selectElement : selectNodes) {
        // 获取“select”元素的各个属性值
        String selectId = selectElement.attributeValue("id");
        String parameterType = selectElement.attributeValue("parameterType");
        String resultType = selectElement.attributeValue("resultType");
        String sqlStatement = selectElement.getText();

        // 输出“select”元素的属性及SQL语句
        System.out.println("相关SQL映射信息：ID=" + selectId + "；参数类型=" + parameterType +
                           "；结果类型=" + resultType + "；SQL语句=" + sqlStatement);
    }
}
```

作用是从一个主配置文件（"mybatis-config-datasource.xml"）中读取到多个mapper资源配置，并逐个加载这些mapper资源文件。针对每个mapper文件，进一步提取出其中所有的SQL `select` 映射定义，包括其ID、参数类型、结果类型以及具体的SQL语句；这里是将这些信息打印出来，后续将保存到实体或者配置方便后续使用。

看下解析的效果：

```shell
正在查看资源：mapper/User_Mapper.xml
相关SQL映射信息：ID=queryUserInfoById；参数类型=java.lang.Long；结果类型=com.xbhog.User；SQL语句=
        SELECT id, userId, userHead, createTime
        FROM user
        where id = #{id}
```

### XML配置构建器

这部分采用`建造者模式`实现XML的配置，该模式比较适合基本物料不变，而其组合经常发生变化的场景；核心目的是将复杂对象的构建过程与它的表示分离。

建造者分为一下几个角色：

1. 抽象建造者：定义了创建产品对象的各个部分的方法（比如零件或组件），一般会有一个方法来获取最终的复杂产品。
2. 具体建造者：实现了抽象建造者接口，完成每个部分的具体构造和装配方法，提供构造过程的具体实现。
3. 产品(复杂对象)：是被构建的复杂对象，包含了多个组成部件。具体建造者创建产品的各个部件并最终组合成完整的产品。

#### 抽象建造者

按照上述的定义以及具体我们想实现的业务可以拆分成两个操作：

1. 初始化XML字符流，转换成Doc供后续使用
2. 解析完的DOC，实体类进行保存，并且需要一个出口进行获得

```java
public abstract class BaseBuilder {

    protected final Configuration configuration;

    public BaseBuilder(Configuration configuration) {
        this.configuration = configuration;
    }

    //只保证外部能够获得所有的信息，具体的配置在子类中赋值
    public Configuration getConfiguration() {
        return configuration;
    }

}
```

BaseBuilder提供了构建Configuration对象所需要的基本方法和属性，并且也对其子类有着通用的构建逻辑。

#### 建造者实现

初始化XML字符流，转换Doc：

```java
public XmlConfigBuilder(Reader reader) {
    //在处理XML配置文件中初始化Configuration
    super(new Configuration());
    // 2. dom4j 处理 xml
    SAXReader saxReader = new SAXReader();
    try {
        Document document = saxReader.read(new InputSource(reader));
        root = document.getRootElement();
    } catch (DocumentException e) {
        e.printStackTrace();
    }
}
```

初始化XML字符流在上一小节已经解决了，在正常Myabtis中的Sql是有占位符或者拼接符；并将相关元素的保存到mappedStatement(映射器语句类)中；

映射类语句类的编写：

```java
/**
 * @author xbhog
 * @describe: 用于封装MyBatis中映射SQL语句的相关信息，包括配置信息、SQL类型、参数类型、结果类型、SQL语句以及动态参数映射等。
 * @date 2024/3/2
 */
public class MappedStatement {

    /**
     * 配置对象，包含MyBatis运行所需的环境、数据库映射等全局配置信息。
     */
    private Configuration configuration;

    /**
     * 映射ID，唯一标识一个MappedStatement，通常对应XML文件中的<mappedStatement>标签的id属性。
     */
    private String id;

    /**
     * SQL命令类型，如INSERT、UPDATE、SELECT或DELETE等。
     */
    private SqlCommandType sqlCommandType;

    /**
     * 参数类型，对应于传入SQL语句的参数类的全限定名。
     */
    private String parameterType;

    /**
     * 结果类型，对应于SQL查询结果映射到的Java类型的全限定名。
     */
    private String resultType;

    /**
     * SQL语句，可能是预编译的静态SQL或带有占位符的动态SQL。
     */
    private String sql;

    /**
     * 动态参数映射集合，键为参数的位置（从0开始计数），值为参数的名称。
     */
    private Map<Integer, String> parameter;

    /**
     * 空构造器，主要用于反射创建实例。
     */
    public MappedStatement() {}

    /**
     * 内部静态嵌套类Builder，遵循建造者设计模式，用于构建MappedStatement实例。
     */
    public static class Builder {

        /**
         * 储存待构建的MappedStatement对象引用。
         */
        private MappedStatement mappedStatement = new MappedStatement();

        /**
         * 初始化Builder对象，并设置MappedStatement的所有必要属性。
         *
         * @param configuration MyBatis的全局配置对象
         * @param id 映射ID
         * @param sqlCommandType SQL命令类型
         * @param parameterType 参数类型全限定名
         * @param resultType 结果类型全限定名
         * @param sql SQL语句
         * @param parameter 动态参数映射集合
         */
        public Builder(Configuration configuration, String id, SqlCommandType sqlCommandType, String parameterType, String resultType, String sql, Map<Integer, String> parameter) {
            mappedStatement.configuration = configuration;
            mappedStatement.id = id;
            mappedStatement.sqlCommandType = sqlCommandType;
            mappedStatement.parameterType = parameterType;
            mappedStatement.resultType = resultType;
            mappedStatement.sql = sql;
            mappedStatement.parameter = parameter;
        }

        /**
         * 完成构建并返回MappedStatement实例，同时进行必要的非空断言检查。
         *
         * @return 已经填充好所有必要属性的MappedStatement实例
         */
        public MappedStatement build() {
            assert mappedStatement.configuration != null : "Configuration must not be null!";
            assert mappedStatement.id != null : "ID must not be null!";
            return mappedStatement;
        }
    }
}
```

存储部分设置完成，我们处理下`select`的操作：

```java
// 遍历所有“mapper”子元素
List<Element> mapperElements = mappers.elements("mapper");
for (Element mapperElement : mapperElements) {
    ......
    //命名空间
    String namespace = mapperRootElement.attributeValue("namespace");
    // 遍历mapper文件中所有的“select”元素(暂时只有这一个操作)
    List<Element> selectNodes = mapperRootElement.elements("select");
    for (Element selectElement : selectNodes) {
        // 获取“select”元素的各个属性值
        String selectId = selectElement.attributeValue("id");
        String parameterType = selectElement.attributeValue("parameterType");
        String resultType = selectElement.attributeValue("resultType");
        String sql = selectElement.getText();

        Map<Integer, String> parameter = new HashMap<>();
        Pattern pattern = Pattern.compile("(#\\{(.*?)})");
        Matcher matcher = pattern.matcher(sql);
        for(int i = 1; matcher.find(); i++){
            String group1 = matcher.group(1);
            String group2 = matcher.group(2);
            log.info("匹配出来的信息为：{}：{}", group1, group2);
            parameter.put(i, group2);
            //替换占位符
            sql = sql.replace(group1,"?");
        }
        //获取全路径
        String msId = namespace + "." + selectId;
        //获取sql方法(select.....)
        String nodeName = selectElement.getName();
        //替换，保持大小写一致
        SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
        //保存
        MappedStatement mappedStatement = new MappedStatement.Builder(configuration, msId,
                sqlCommandType, parameterType, resultType, sql, parameter).build();
        //配置文件设置()
        configuration.addMappedStatement(mappedStatement);
    }
    // 注册Mapper映射器
    configuration.addMapper(Class.forName(namespace));
}
```

可以发现上述的代码中出来在XML的解析以及映射器属性的保存外，还有新加的configuration类。在mybatis中Configuration是一个重量级的核心配置类，几乎包含了Mybatis运行的所有的配置信息。

在正式Mybatis中Configuration的作用：

1. 存储全局配置信息：如数据库连接信息（driver、url、username、password）、事务管理器设置、映射文件位置、自定义类型处理器、日志工厂等。
2. **管理映射资源：存储并管理所有 `<mapper>` 标签对应的 `MappedStatement` 对象，这些对象包含了SQL语句、参数类型、结果类型、动态SQL解析器等信息。(本次处理的重点)**
3. **提供动态SQL解析：基于`Configuration`对象，MyBatis能解析包含动态元素的SQL语句。(本次处理的重点)**
4. 支持延迟加载和缓存策略：在`Configuration`中可以配置二级缓存、全局的缓存开关、延迟加载策略等。
5. 提供内置对象工厂：`Configuration`对象维护了一个对象工厂，用于在运行时创建诸如ParameterHandler、ResultSetHandler、StatementHandler等核心对象。

**本节处理流程：**

![img](https://xbhog-img.oss-cn-hangzhou.aliyuncs.com/2022/20240302220357.png)

这得看下`SqlSessionFatoryBuilder`入口类的实现：

```java
public class SqlSessionFactoryBuilder {

    public SqlSessionFactory build(Reader reader) {
        XMLConfigBuilder xmlConfigBuilder = new XMLConfigBuilder(reader);
        return build(xmlConfigBuilder.parse());
    }

    public SqlSessionFactory build(Configuration config) {
        return new DefaultSqlSessionFactory(config);
    }
}
```

**先说结论：configuration类本身不是单例类，相反是Mybatis通过** **SqlSessionFactoryBuilder** **来确保整个应用中只有一个 Configuration 实例，Configuration在****SqlSessionFactoryBuilder构建阶段完成实例化的操作。**

一旦 `SqlSessionFactory` 被创建，`SqlSessionFatoryBuilder` 的使命就完成了，之后 `SqlSessionFactory` 就会被用来创建 `SqlSession` 实例，进而执行 SQL 语句、管理事务和进行其他数据库操作。由于 `SqlSessionFatoryBuilder` 的任务完成后就可以丢弃，因此通常采取即用即抛的原则，正是由于`SqlSessionFatoryBuilder` 的这种原则，保证了Configuration 有了单例的特性。

### Mapper文件中方法的执行操作封装

看下之前的invoke方法的实现：

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if(Object.class.equals(method.getDeclaringClass())){
        return method.invoke(this,args);
    }else{
        //todo 具体接口实现的方式
        return sqlSession.selectOne(method.getName(), args);
    }
}
```

在上一局中我们在invoke方法中指定了sql的处理类型，基本操作都知道肯定不会有一种类型；为了读者能更好的理解，这里先看下上一局和这一局代码的新旧流程对比。

对比范围：

```txt
└─src 
  ├─main 
  │ └─java 
  │   └─com 
  │     └─xbhog 
  │       ├─binding 
  │       │ ├─MapperMethod.java(新增) 
  │       │ ├─MapperProxy.java 
  │       │ ├─MapperProxyFactory.java 
  │       │ └─MapperRegistry.java 
```



该流程处理代码流程不变，细节地方会有Configuration处理，详细请看代码操作。

![MapperRegistry_getMapper](..\img\MapperRegistry_getMapper.jpg)

变化比较大的是在`MapperProxy`中`invoke`方法的执行操作上；

![MapperProxy_invoke](..\img\MapperProxy_invoke.jpg)

这个时序图是MapperMethod类流程；

简单介绍下功能

1. 在mapperMethod初始化前会先从methodCache中进行查询，存在返回，不存在构建完并返回，类似于缓存。
2. mapperMethod初始化内包含了SqlCommand指令的构建，采用的模式还是建造者模式，内置方法名和方法类型属性，通过接口名和方法名从Configuration获取mappedStatements构建属性值。
3. 其他操作就是把crud进行分开判断并执行。

到此所有涉及到的操作，都或多或少的写到了，接下来进行测试。

## 测试

环境构建

```txt
│  └─test
│      ├─java
│      │  └─com
│      │      └─xbhog
│      │              AppTest.java
│      │              IUserDao.java
│      │              User.java
│      │              
│      └─resources
│          │  mybatis-config-datasource.xml
│          │  
│          └─mapper
│                  User_Mapper.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>

    <mappers>
        <mapper resource="mapper/User_Mapper.xml"/>
    </mappers>

</configuration>
```

```xml
<mapper namespace="com.xbhog.IUserDao">

    <select id="queryUserInfoById" parameterType="java.lang.Long" resultType="com.xbhog.User">
        SELECT id, userId, userHead, createTime
        FROM user
        where id = #{id}
    </select>

</mapper>
```

```java
public class AppTest extends TestCase {

    private Logger logger = LoggerFactory.getLogger(AppTest.class);
    /**
     * Rigourous Test :-)
     */
    public void testApp() throws Exception {
        // 1. 从SqlSessionFactory中获取SqlSession
        Reader reader = ResourceUtil.getUtf8Reader("mybatis-config-datasource.xml");
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
        SqlSession sqlSession = sqlSessionFactory.openSession();

        // 2. 获取映射器对象
        IUserDao userDao = sqlSession.getMapper(IUserDao.class);

        // 3. 测试验证
        String res = userDao.queryUserInfoById("10001");
        logger.info("测试结果：{}", res);
    }
}
```

结果如下：

```sh
正在查看资源：mapper/User_Mapper.xml
11:55:58.436 [main] INFO  c.xbhog.builder.xml.XmlConfigBuilder - 匹配出来的信息为：#{id}：id
11:55:58.456 [main] INFO  com.xbhog.AppTest - 测试结果：你被代理了！方法：com.xbhog.IUserDao.queryUserInfoById 入参：[Ljava.lang.Object;@71c7db30,待执行的SQl:
        SELECT id, userId, userHead, createTime
        FROM user
        where id = ?
```

## 总结

整个流程大致是从配置文件出发，经过 `SqlSessionFactoryBuilder` 和 `XmlConfigBuilder` 构建并填充 `Configuration`；然后 `Configuration` 被用来创建 `SqlSessionFactory`，进而生产 `SqlSession`；同时，`Configuration` 中还维护了所有 `MappedStatement` 以及 `Mapper` 相关的注册信息，确保 `SqlSession` 在处理数据库操作请求时能够找到正确的映射关系并执行相应的方法。对于 `Mapper` 接口，它们通过 `MapperRegistry`、`MapperProxyFactory` 和 `MapperProxy` 构成了一个面向接口编程的持久层访问机制。

| **类名**                     | **类的作用**                                                 |
| ---------------------------- | ------------------------------------------------------------ |
| **SqlSessionFactoryBuilder** | 这是一个工具类，用于从配置信息（如XML配置文件或预定义的Configuration对象）构建`SqlSessionFactory`实例。它接收配置输入，解析并验证配置，然后创建并初始化`SqlSessionFactory`。 |
| **SqlSessionFactory**        | 可以理解为SqlSession的工厂，它持有MyBatis的核心配置信息（由`Configuration`对象提供）。每个MyBatis应用的核心就是`SqlSessionFactory`实例，它负责创建`SqlSession`对象。`SqlSessionFactory`通常是线程安全的，可以被多个线程共享，并在整个应用生命周期内保持。 |
| **SqlSession**               | 是MyBatis执行数据库操作的主要入口点，它提供了CRUD（增删改查）等各种数据库操作方法。每个SqlSession对象都与一个数据库连接（Connection）关联，且不是线程安全的，通常在一个请求或操作范围内使用。 |
| **Configuration**            | 这是MyBatis的全局配置容器，它存储了所有关于MyBatis的行为配置、所有映射器的注册信息、数据源、事务管理器、类型处理器等配置。`SqlSessionFactoryBuilder`正是根据`Configuration`的信息来构建`SqlSessionFactory`。 |
| **DefaultSqlSessionFactory** | 这是`SqlSessionFactory`的一个具体实现类，继承自`SqlSessionFactory`接口，它包含了创建和管理`SqlSession`的实际逻辑。 |
| **DefaultSqlSession**        | 这是`SqlSession`接口的默认实现，它封装了对数据库的CRUD操作以及对`MappedStatement`的访问。 |
| **SqlCommandType**           | 表示SQL命令的类型枚举，包括INSERT、UPDATE、DELETE、SELECT等，这是在`MappedStatement`中用来区分不同类型的SQL操作。 |
| **MappedStatement**          | 表示一个已经映射好的SQL语句及其相关配置，包括SQL类型、参数类型、结果类型、SQL语句本身以及可能的动态SQL节点和参数映射规则。 |
| **XmlConfigBuilder**         | 用于解析MyBatis的XML配置文件，将XML配置信息转换为`Configuration`对象。 |
| **BaseBuilder**              | 在MyBatis中，如果有多个Builder类有着相似的构建逻辑，可能会定义一个基类`BaseBuilder`，提取公共方法和属性，不过这里并未明确提到它在MyBatis中的具体实现。 |
| **MapperRegistry**           | 在MyBatis中，MapperRegistry是一个注册和管理所有`Mapper`接口的地方，它负责存储和查找`Mapper`接口对应的`MapperProxy`和`MapperMethod`。 |
| **MapperMethod**             | 表示在`Mapper`接口中定义的某个方法的具体实现，它封装了方法对应的SQL执行逻辑。 |
| **MapperProxy**              | MyBatis利用Java动态代理机制，通过`MapperProxy`来实现代理`Mapper`接口的对象，当调用`Mapper`接口方法时，实际上是调用了`MapperProxy`中的方法，从而执行SQL操作。 |
| **MapperProxyFactory**       | 负责创建`MapperProxy`实例的工厂类，它根据`Mapper`接口生成对应的动态代理对象，使用户可以通过接口的方式来执行数据库操作，而不是直接与`SqlSession`打交道。每次需要新的`Mapper`代理对象时，都会通过`MapperProxyFactory`来创建。 |

