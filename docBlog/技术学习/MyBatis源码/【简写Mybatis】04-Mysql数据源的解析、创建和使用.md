## 前言

**在学习MyBatis源码文章中，斗胆想将其讲明白；故有此文章，如有问题，不吝指教！**

**注意：**

> 学习源码一定一定不要太关注代码的编写，而是注意代码实现思想；
>
> 通过设问方式来体现代码中的思想；方法：**5W+1H**

**源代码：**[**https://gitee.com/xbhog/mybatis-xbhog**](https://gitee.com/xbhog/mybatis-xbhog)**；**[**https://github.com/xbhog/mybatis-xbhog**](https://github.com/xbhog/mybatis-xbhog)**；交个朋友，欢迎star。**
也可关注笔者公众号：
![](https://img-blog.csdnimg.cn/img_convert/9d27a752dec20601bfd3fc7938739419.jpeg)

## 回顾&分析

在上一局的实现中[【简写Mybatis】03-Mapper xml的注册和使用](https://blog.csdn.net/weixin_43908900/article/details/136586127)；我们完善了XML的解析、读取、封装；新增Configuration配置类进行数据流转，最后对Mapper select方法进行了封装等操作；**但最后还是没有数据库的执行操作；**

本章节就不看测试类，来看下上一局的测试输出。

```java
正在查看资源：mapper/User_Mapper.xml
18:49:23.642 [main] INFO  c.xbhog.builder.xml.XmlConfigBuilder - 匹配出来的信息为：#{id}：id
18:49:23.668 [main] DEBUG c.x.binding.MapperMethod$SqlCommand - 查看当前方法【SqlCommand】参数传进来的信息：接口信息：interface com.xbhog.IUserDao，方法信息：public abstract java.lang.String com.xbhog.IUserDao.queryUserInfoById(java.lang.String)
18:49:23.670 [main] INFO  com.xbhog.AppTest - 测试结果：你被代理了！方法：com.xbhog.IUserDao.queryUserInfoById 入参：[Ljava.lang.Object;@71c7db30,待执行的SQl:
        SELECT id, userId, userHead, createTime
        FROM user
        where id = ?
```

上述测试用例只输出了SQL没有执行；再看下实际Mybatis中数据库使用；前置条件是需要配置相关的数据库信息；

```xml
 <!--连接数据库环境-->
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value=" jdbc:mysql://xxx.xxx:3306/mybatis"/>
                <property name="username" value="mybatis"/>
                <property name="password" value="123456"/>
            </dataSource>
        </environment>
    </environments>
```

## 目的

1. 解析数据库相关数据(XML解析)
2. 数据源的设置与使用
3. 初步对 SQL 的执行结果进行简单包装
4. 事务管理(**扩展**)

## 实现

项目结构如下： 

```txt
├─src
│  ├─main
│  │  └─java
│  │      └─com
│  │          └─xbhog
│  │              ├─binding
│  │              │      MapperMethod.java
│  │              │      MapperProxy.java
│  │              │      MapperProxyFactory.java
│  │              │      MapperRegistry.java
│  │              │      
│  │              ├─builder
│  │              │  │  BaseBuilder.java
│  │              │  │  
│  │              │  └─xml
│  │              │          XmlConfigBuilder.java
│  │              │          
│  │              ├─datasource
│  │              │      DataSourceFactory.java
│  │              │      DruidDataSourceFactory.java
│  │              │      
│  │              ├─mapping
│  │              │      Environment.java
│  │              │      MappedStatement.java
│  │              │      SqlCommandType.java
│  │              │      
│  │              ├─session
│  │              │  │  Configuration.java
│  │              │  │  SqlSession.java
│  │              │  │  SqlSessionFactory.java
│  │              │  │  SqlSessionFactoryBuilder.java
│  │              │  │  TransactionIsolationLevel.java
│  │              │  │  
│  │              │  └─defaults
│  │              │          DefaultSqlSession.java
│  │              │          DefaultSqlSessionFactory.java
│  │              │          
│  │              └─transaction
│  │                  │  Transaction.java
│  │                  │  TransactionFactory.java
│  │                  │  
│  │                  └─jdbc
│  │                          JdbcTransaction.java
│  │                          JdbcTransactionFactory.java
│  │                          
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



### 数据库信息的解析

跟过文章的朋友应该清楚Mybatis中文件配置解析是在哪个地方实现;


这里简单提一下路径：

`\mybatis-simple-04\src\main\java\com\xbhog\builder\xml\XmlConfigBuilder.java`

`parse`方法中实现;

看下读取操作：

```java
public void parseXml() throws DocumentException {
        // xml文件内容转换为字符串流
        Reader reader = ResourceUtil.getUtf8Reader("mybatis-config-datasource.xml");

        // 从字符串流中读取并创建XML Document对象
        SAXReader saxReader = new SAXReader();
        Document document = saxReader.read(new InputSource(reader));

        // 获取XML文档的根元素
        Element rootElement = document.getRootElement();
        Element environments = rootElement.element("environments");

        //获取默认的环境信息
        String envDefault = environments.attributeValue("default");
        //获取环境所有的信息
        List<Element> elements = environments.elements("environment");
        for(Element element : elements){
            String id = element.attributeValue("id");
            if(!Objects.equals(id,envDefault)){
                return;
            }
            //获取事务类型
            String transactionType = element.element("transactionManager").attributeValue("type");
            //获取数据源类型
            Element dataSource = element.element("dataSource");
            String dataSourceType = dataSource.attributeValue("type");
            //参数列表
            Properties properties = new Properties();
            List<Element> propertys = dataSource.elements("property");
            for(Element property : propertys){
                properties.setProperty(property.attributeValue("name"),property.attributeValue("value"));
            }
            logger.info("事务类型：{}，数据源类型：{},数据源参数列表：{}",transactionType,dataSourceType,JSONUtil.parse(properties));
        }

    }
```

结果测试如下：

```shell
Connected to the target VM, address: '127.0.0.1:62810', transport: 'socket'
16:25:38.000 [main] INFO  com.xbhog.AppTest - 事务类型：JDBC，数据源类型：DRUID,数据源参数列表：{"url":"jdbc:mysql://127.0.0.1:3306/dev?useUnicode=true","password":"123456","driver":"com.mysql.jdbc.Driver","username":"root"}

Disconnected from the target VM, address: '127.0.0.1:62810', transport: 'socket'
```

有效数据拿到了，接下来进行属性的映射以及数据流转。

既然是对数据库的操作，那么避免不了数据源、事务的设置。

## 设置数据源

设置DataSourceFactory来创建和初始化数据源，这里采用工厂模式实现，使得数据源的实例化过程与MyBatis核心逻辑解耦，增强了程序的灵活性和可扩展性；并且工厂可以生产多种类型的数据源，既满足职责分离也具有很强的扩展。

既然是创建和初始化数据源，那么需要的两个方法；

1. 属性装配(赋值)
2. 生成数据源并获取

代码如下：

```java
public class DruidDataSourceFactory implements DataSourceFactory {

    private Properties props;

    @Override
    public void setProperties(Properties props) {
        this.props = props;
    }

    @Override
    public DataSource getDataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setDriverClassName(props.getProperty("driver"));
        dataSource.setUrl(props.getProperty("url"));
        dataSource.setUsername(props.getProperty("username"));
        dataSource.setPassword(props.getProperty("password"));
        return dataSource;
    }

}
```

这里串联到数据源环境解析的XmlConfigBuilder中。

**遍历xml中`<property>`标签数据赋值到Spring中Properties类，为了后续的使用，将env信息进行实体保存并放到Configuration中，用于数据流转。**

代码如下：

```java
private void environmentElement(Element environments) throws Exception{
        String env= environments.attributeValue("default");
        List<Element> elements = environments.elements("environment");
        for(Element e : elements){
            String id = e.attributeValue("id");
            if(env.equals(id)){
                Element dataSourceElement = e.element("dataSource");
                //获取数据库连接池
                DruidDataSourceFactory dataSourceFactory = new DruidDataSourceFactory();
                List<Element> propertyList = dataSourceElement.elements("property");
                //属性配置项
                Properties props = new Properties();
                for(Element property : propertyList){
                    props.setProperty(property.attributeValue("name"), property.attributeValue("value"));
                }
                dataSourceFactory.setProperties(props);
                DataSource dataSource = dataSourceFactory.getDataSource();

                //构建环境信息
                Environment.Builder environmentBuilder = new Environment.Builder(id).dataSource(dataSource);
                configuration.setEnvironment(environmentBuilder.build());
            }
        }
    }
```

## SQL封装

上述操作都是数据准备阶段，完成后通过数据源进行sql执行并进行结果包装。在select方法的实现中通过Configuation获取数据源的连接方法，通过查询参数的设置以及sql的执行获取返回结果。

```java
MappedStatement mappedStatements = configuration.getMappedStatements(statement);
Environment environment = configuration.getEnvironment();
Connection connection = environment.getDataSource().getConnection();
String sql = mappedStatements.getSql();
PreparedStatement preparedStatement = connection.prepareStatement(sql);
preparedStatement.setLong(1,Long.parseLong(((Object[]) parameter)[0].toString()));
ResultSet resultSet = preparedStatement.executeQuery();
```

返回结果的处理；

入参方法解释：对数据库结果数据进行清洗;入参jdbc执行结果以及结果类型(对应于SQL查询结果映射到的Java类型的全限定名-User)

```java
List<T> resultSet2Obj = resultSet2Obj(resultSet, Class.forName(mappedStatements.getResultType()));
```

对SQL结果的处理：

1. 获取SQL结果MeatData数据
2. 获取字段数量
3. 类属性处理以及实体反射赋值

代码如下：

```java
try {
    ResultSetMetaData metaData = resultSet.getMetaData();
    int columnCount = metaData.getColumnCount();
    // 每次遍历行值
    while (resultSet.next()) {
        T obj = (T) clazz.newInstance();
        for (int i = 1; i <= columnCount; i++) {
            Object value = resultSet.getObject(i);
            String columnName = metaData.getColumnName(i);
            //属性首字母大写
            String setMethod = "set" + columnName.substring(0, 1).toUpperCase() + columnName.substring(1);
            Method method;
            if (value instanceof Timestamp) {
                method = clazz.getMethod(setMethod, Date.class);
            } else {
                //会寻找clazz类(User)中名字为setMethod（setId），并且只接受一个参数的方法，且该参数类型与value.getClass()所返回的类型一致
                method = clazz.getMethod(setMethod, value.getClass());
            }
            //实体类属性赋值
            method.invoke(obj, value);
        }
        list.add(obj);
    }
} catch (Exception e) {
    e.printStackTrace();
}
```

## 事务处理

在平时接触最多的数据库连接就是JDBC，需要明确的是数据库连接应该在事务中处理，这样才能保证数据的提交、回滚、关闭的原子性，在代码实现上也是如此。

不过一般我们并不会使用 Mybatis 管理事务，而是将 Mybatis 集成到 Spring，由 Spring 进行事务的管理。

事务的类型也有几个，这里先处理JDBC的事务，同数据源一样开工厂模式进行封装。

JDBC 事务直接使用JDBC的commit、rollback，依赖于数据源获得的连接来管理事务范围。

代码如下：

```java
public class JdbcTransaction implements Transaction {

    protected Connection connection;

    protected DataSource dataSource;

    protected boolean autoCommit;

    protected TransactionIsolationLevel level = TransactionIsolationLevel.NONE;


    public JdbcTransaction(DataSource dataSource, boolean autoCommit, TransactionIsolationLevel level) {
        this.dataSource = dataSource;
        this.autoCommit = autoCommit;
        this.level = level;
    }

    public JdbcTransaction(Connection connection) {
        this.connection = connection;
    }

    @Override
    public Connection getConnection() throws SQLException {
        //获取一个新的数据库连接实例
        connection = dataSource.getConnection();
        //设置当前数据库连接的事务隔离级别
        connection.setTransactionIsolation(level.getLevel());
        //控制数据库连接的自动提交行为
        connection.setAutoCommit(autoCommit);
        return connection;
    }

    @Override
    public void commit() throws SQLException {
        if(Objects.nonNull(connection) && !connection.getAutoCommit()){
            connection.commit();
        }
    }

    @Override
    public void rollback() throws SQLException {
        if(Objects.nonNull(connection) && !connection.getAutoCommit()){
            connection.rollback();
        }
    }

    @Override
    public void close() throws SQLException {
        if (connection != null && !connection.getAutoCommit()) {
            connection.close();
        }
    }
}
```

## 测试

新建User表,user_mapper.xml对照：

```xml
<select id="queryUserInfoById" parameterType="java.lang.Long" resultType="com.xbhog.User">
        SELECT id, userId, userHead
        FROM user
        where id = #{id}
    </select>
```

```java
public void testApp() throws Exception {
        // 1. 从SqlSessionFactory中获取SqlSession
        Reader reader = ResourceUtil.getUtf8Reader("mybatis-config-datasource.xml");
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
        SqlSession sqlSession = sqlSessionFactory.openSession();

        // 2. 获取映射器对象
        IUserDao userDao = sqlSession.getMapper(IUserDao.class);

        // 3. 测试验证
        User user = userDao.queryUserInfoById(1L);
        logger.info("测试结果：{}", JSONUtil.parse(user));
    }
```

```shell
11:33:24.257 [main] INFO  com.xbhog.AppTest - 测试结果：{"userId":"10001","userHead":"1_04","id":1}
```

## 学习&参考

https://mp.weixin.qq.com/s/g9cZOsYSFJtp6FoFtMg6RQ

mybatis源码

AI大模型