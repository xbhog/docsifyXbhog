## 前言

**注意：**

> 学习源码一定一定不要太关注代码的编写，而是注意代码实现思想：
>
> 通过设问方式来体现代码中的思想；方法：**5W+1H**

**源代码：**[**https://gitee.com/xbhog/mybatis-xbhog**](https://gitee.com/xbhog/mybatis-xbhog)**；**[**https://github.com/xbhog/mybatis-xbhog**](https://github.com/xbhog/mybatis-xbhog)**；交个朋友，有价值欢迎star。**

## 回顾&分析

上一局实现Mapper接口和映射器通过代理类的方式进行链接。

上一局测试类：[【简写MyBatis】01-简单映射器](https://blog.csdn.net/weixin_43908900/article/details/136131481?spm=1001.2014.3001.5501)；虽然基本功能实现了，但是还不智能，可以发现该测试类中的映射器代理工厂只能实现单一的接口代理，SqlSession也需要规范化处理；将映射器代理和方法的调用类似服务进行包装。

```java
@Test
public void test_MapperProxyFactory() {
    MapperProxyFactory<IUserDao> factory = new MapperProxyFactory<>(IUserDao.class);

    Map<String, String> sqlSession = new HashMap<>();
    sqlSession.put("cn.bugstack.mybatis.test.dao.IUserDao.queryUserName", "模拟执行 Mapper.xml 中 SQL 语句的操作：查询用户姓名");
    sqlSession.put("cn.bugstack.mybatis.test.dao.IUserDao.queryUserAge", "模拟执行 Mapper.xml 中 SQL 语句的操作：查询用户年龄");
    IUserDao userDao = factory.newInstance(sqlSession);

    String res = userDao.queryUserName("10001");
    logger.info("测试结果：{}", res);
}
```

## 目的

1. 根据包路径实现接口的扫描和注册
2. SqlSession规范化处理

## 实现

项目结构：

```java
└─src 
  ├─main 
  │ └─java 
  │   └─com 
  │     └─xbhog 
  │       ├─binding 
  │       │ ├─MapperProxy.java 
  │       │ ├─MapperProxyFactory.java 
  │       │ └─MapperRegistry.java 
  │       └─session 
  │         ├─DefaultSqlSession.java 
  │         ├─DefaultSqlSessionFactory.java 
  │         ├─SqlSession.java 
  │         └─SqlSessionFactory.java 
  └─test 
    └─java 
      └─com 
        └─xbhog 
          ├─AppTest.java 
          └─IUserDao.java 
```

项目类图

![img](https://xbhog-img.oss-cn-hangzhou.aliyuncs.com/2022/20240224221752.png)

### 根据包路径实现接口的扫描和注册

可以通过自定义MapperRegistry这个类实现该功能，当然你也可以叫其他的名字(zhangsan、lisi😅)；我们只需要将上一局的MapperProxyFactory封装到MapperRegistry，把需要扫描和注册的接口保存到Map中进行预处理，最后代理进行随时使用就可以了；铺垫结束，开始上代码。

先扫描包下所有的Class类，然后保存到Map中。

```java
package com.xbhog;

import cn.hutool.core.lang.ClassScanner;

import java.util.HashMap;
import java.util.Map;
import java.util.Set;

/**
 * @author xbhog
 * @describe: 接口注册器
 * @date 2024/2/25
 */
public class MapperRegistry {

    private final Map<Class<?>,MapperProxyFactory<?>> interfaceMaps = new HashMap<>();
    public void addMapper(String packageName){
        Set<Class<?>> scanPackage = ClassScanner.scanPackage(packageName);
        scanPackage.forEach(clazz -> {
            addMappers(clazz);
        });
    }

    private void addMappers(Class<?> clazz) {
        if(clazz.isInterface()){
            //判断是否重复添加
            if(haveInterface(clazz)){
                throw new RuntimeException("Type " + clazz + " is already known to the MapperRegistry.");
            }
        }
        // 注册映射器代理工厂
        interfaceMaps.put(clazz, new MapperProxyFactory<>(clazz));
    }

    private boolean haveInterface(Class<?> clazz) {
        return interfaceMaps.containsKey(clazz);
    }

}
```



然后将上一局的接口和代理工厂操作封装进方法中。

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession){
    MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) interfaceMaps.get(type);
    if(Objects.isNull(mapperProxyFactory)){
        throw new RuntimeException("Type " + type + " is not known to the MapperRegistry.");
    }
    return (T)mapperProxyFactory.newInstance(sqlSession);
}
```

### SqlSession规范化处理

先定义一个执行Sql、获取映射器的标准接口：

```java
/**
 * @author xbhog
 * @describe: 定义简单的Mapper操作方法
 * @date 2024/2/25
 */
public interface SqlSession {

    <T> T selectOne(String statement,Object parameter);

    /**
     *得到接口映射器
     * @param type 接口类型
     * @return
     */
    <T> T getMapper(Class<T> type);
}
```

接口实现方式：

```java
package com.xbhog.session;

import com.xbhog.binding.MapperRegistry;

/**
 * @author xbhog
 * @describe:
 * @date 2024/2/25
 */
public class DefaultSqlSession implements SqlSession{

    private MapperRegistry mapperRegistry;

    public DefaultSqlSession(MapperRegistry mapperRegistry) {
        this.mapperRegistry = mapperRegistry;
    }

    @Override
    public <T> T  selectOne(String statement,Object parameter) {
        return (T) ("你被代理了！" + "方法：" + statement + " 入参：" + parameter);
    }

    @Override
    public <T> T getMapper(Class<T> type) {
        return mapperRegistry.getMapper(type,this);
    }
}
```

测试一下：

```java
package com.xbhog;

import com.xbhog.binding.MapperProxyFactory;
import com.xbhog.binding.MapperRegistry;
import com.xbhog.session.DefaultSqlSession;
import junit.framework.Test;
import junit.framework.TestCase;
import junit.framework.TestSuite;

import java.util.HashMap;
import java.util.Map;

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
        DefaultSqlSession sqlSession = new DefaultSqlSession(mapperRegistry);
        IUserDao user = sqlSession.getMapper(IUserDao.class);
        String userName = user.getUserName("xbhog");
        System.out.println("输出的信息："+userName);
    }
}
```

![img](https://xbhog-img.oss-cn-hangzhou.aliyuncs.com/2022/20240225145343.png)

到这里其实已经可以满足需求了，但是看了下源码发现还是不行，它最外层又封装了一层代理工厂；应该是为了后续的代码扩展，简单工厂模式有助于代码的模块性和可维护性，功能上后续会有配置管理、资源管理、执行器选择和插件等需求；走一步看三步的老狐狸(┬┬﹏┬┬)。先抄作业。

```java
package com.xbhog.session;

import com.xbhog.binding.MapperRegistry;

/**
 * @author xbhog
 * @describe:
 * @date 2024/2/25
 */
public class DefaultSqlSessionFactory implements SqlSessionFactory{

    private final MapperRegistry mapperRegistry;

    public DefaultSqlSessionFactory(MapperRegistry mapperRegistry) {
        this.mapperRegistry = mapperRegistry;
    }

    @Override
    public SqlSession openSession() {
        return new DefaultSqlSession(mapperRegistry);
    }
}
```

## 测试

```java
package com.xbhog;

import com.xbhog.binding.MapperProxyFactory;
import com.xbhog.binding.MapperRegistry;
import com.xbhog.session.DefaultSqlSession;
import com.xbhog.session.DefaultSqlSessionFactory;
import com.xbhog.session.SqlSession;
import com.xbhog.session.SqlSessionFactory;
import junit.framework.Test;
import junit.framework.TestCase;
import junit.framework.TestSuite;

import java.util.HashMap;
import java.util.Map;

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

![img](https://xbhog-img.oss-cn-hangzhou.aliyuncs.com/2022/20240225145343.png)

## 总结

1. **What（什么）**：

- MapperRegistry是一个注册表，用于存储映射器接口（Mapper Interface）和对应的MapperProxyFactory。它负责管理映射器接口的生命周期。
- DefaultSqlSessionFactory是MyBatis框架中用于创建SqlSession的工厂类。SqlSession是MyBatis的核心接口，用于执行SQL命令和获取映射结果。

1. **Why（为什么）**：

- MapperRegistry的存在是为了确保映射器接口能够被MyBatis框架识别和管理，以便在运行时为这些接口创建代理对象，实现数据库操作的动态绑定。
- DefaultSqlSessionFactory的目的是为了提供一个统一的入口点，用于创建和管理SqlSession实例。这样可以保证SqlSession的创建和关闭遵循统一的规范，同时提供了会话管理的能力。

1. **Who（谁）**：

- MapperRegistry的使用者是MyBatis框架自身，它内部使用MapperRegistry来处理映射器接口的注册和代理对象的创建。
- DefaultSqlSessionFactory的使用者是应用程序的开发者，他们通过SqlSessionFactory来获取SqlSession实例，进而执行数据库操作。

1. **Where（在哪里）**：

- MapperRegistry是MyBatis框架的一部分，通常在MyBatis配置初始化时创建，并在整个应用程序的生命周期中存在。
- DefaultSqlSessionFactory通常在应用程序启动时创建，并保存在一个全局的变量中，以便在需要时获取SqlSession实例。

1. **When（何时）**：

- MapperRegistry的注册发生在MyBatis应用程序启动时，特别是在构建SqlSessionFactory的过程中。
- DefaultSqlSessionFactory的创建也是在应用程序启动时，通常是在初始化阶段，用于后续的数据库操作。

1. **How（如何）**：

- MapperRegistry通过扫描指定包下的映射器接口，并将它们与对应的MapperProxyFactory关联起来。当需要执行映射器接口中的方法时，MapperRegistry会使用MapperProxyFactory来创建一个MapperProxy代理对象。
- DefaultSqlSessionFactory通过解析MyBatis的配置文件（如mybatis-config.xml，**下一节的操作**）来创建。它提供了openSession()方法，用于创建SqlSession实例。开发者可以通过SqlSession实例来执行映射器接口中定义的数据库操作。

**需要注意的是：通过这两节可以看到mybatis中运用了大量的工厂模式；对外提供统一的方法，屏蔽细节以及上下文的关联关系，最终目的服务于用户，简化使用。**

## 参考

https://mp.weixin.qq.com/s/o6lnWJqU_6FNO8HpxAs9gA

ChatGPT问答