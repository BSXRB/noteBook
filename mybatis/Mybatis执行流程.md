# Mybatis执行流程

## Mybatis使用流**程**

1. **配置xml文件**

   ```xml
   <?xml version="1.0" encoding="UTF-8" ?>
   <!DOCTYPE configuration
           PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
           "http://mybatis.org/dtd/mybatis-3-config.dtd">
   <!-- 根标签 -->
   <configuration>
       <properties>
           <property name="driver" value="com.mysql.jdbc.Driver"/>
           <property name="url" value="jdbc:mysql://127.0.0.1:3306/mybatisread?useUnicode=true&amp;characterEncoding=utf-8&amp;allowMultiQueries=true&amp;serverTimezone=UTC"/>
           <property name="username" value="root"/>
           <property name="password" value="root"/>
       </properties>
       <settings>
           <setting name="mapUnderscoreToCamelCase" value="true"/>
           <!--setting name="localCacheScope" value="STATEMENT"/-->
       </settings>
       <!-- 环境，可以配置多个，default：指定采用哪个环境 -->
       <environments default="test">
           <!-- id：唯一标识 -->
           <environment id="test">
               <!-- 事务管理器，JDBC类型的事务管理器 -->
               <transactionManager type="JDBC" />
               <!-- 数据源，池类型的数据源 -->
               <dataSource type="POOLED">
                   <property name="driver" value="com.mysql.jdbc.Driver" />
                   <property name="url" value="jdbc:mysql://127.0.0.1:3306/mybatisread?serverTimezone=UTC" />
                   <property name="username" value="root" />
                   <property name="password" value="root" />
               </dataSource>
           </environment>
           <environment id="development">
               <!-- 事务管理器，JDBC类型的事务管理器 -->
               <transactionManager type="JDBC" />
               <!-- 数据源，池类型的数据源 -->
               <dataSource type="POOLED">
                   <property name="driver" value="${driver}" /> <!-- 配置了properties，所以可以直接引用 -->
                   <property name="url" value="${url}" />
                   <property name="username" value="${username}" />
                   <property name="password" value="${password}" />
               </dataSource>
           </environment>
       </environments>
       <mappers>
           <mapper resource="mapper/gmapper.xml" />
       </mappers>
   </configuration>
   ```

   

2. **创建用户类**

```java
package dto;

/**
 * @Classname User
 * @Description TODO
 * @Date 2020/9/1 10:14
 * @Created by rou
 */
public class User {
    private String userName;
    private int id;
    private String gender;
    private int age;

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getGender() {
        return gender;
    }

    public void setGender(String gender) {
        this.gender = gender;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "userName='" + userName + '\'' +
                ", id=" + id +
                ", gender='" + gender + '\'' +
                ", age=" + age +
                '}';
    }
}

```

3. **写sql语句**

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!-- mapper:根标签，namespace：命名空间，随便写，一般保证命名空间唯一 -->
<mapper namespace="TestMapper">
    <!-- statement，内容：sql语句。id：唯一标识，随便写，在同一个命名空间下保持唯一
       resultType：sql语句查询结果集的封装类型,tb_user即为数据库中的表
     -->
    <select id="queryUser" resultType="dto.User">
        select * from user where id=#{id}
    </select>
    <insert id="insertUser" parameterType="dto.User">
        insert into user(user_name,gender,age)values(#{userName},#{gender},#{age})
    </insert>



</mapper>
```

4.**编写接口**

```java
import dto.User;
import org.apache.ibatis.annotations.Param;

/**
 * @Classname TestMapper
 * @Description TODO
 * @Date 2020/9/1 10:17
 * @Created by rou
 */
public interface TestMapper {
    User queryUser(@Param("id")int id);
    int insertUser(User user);
}

```

5.**demo**

```java
import java.io.IOException;
import java.io.InputStream;

import dto.User;
import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

/**
 * @Classname Main
 * @Description TODO
 * @Date 2020/9/1 10:23
 * @Created by rou
 */
public class Main {
    public static void main(String[] args) throws Exception {
        TestMapper testMapper;
        String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        SqlSession sqlSession = sqlSessionFactory.openSession(true);
        testMapper = sqlSession.getMapper(TestMapper.class);
        User user = new User();
        user.setId(1);
        user.setAge(25);
        user.setGender("男");
        user.setUserName("llc");
        testMapper.insertUser(user);
        System.out.println(testMapper.queryUser(1));
    }
}

```

对比直接使用jdbc，mybatis无论是从功能上还是开发难易程度上都有很明显的优势，但是在使用的过程中对于从xml读取sql语句到通过接口执行的详细过程一直不是很了解。称现在有时间刚好可以看看mybatis的源码解决这个困惑了很久的问题。

## 源码阅读

虽然sqlSession中有关于增删改查的所有api，但是mybatis还是建议我们使用mapper，那么mapper是如何与sqlSession关联的，这就需要我们找到mapper的实现类在哪里创建的。创建方法最终指向了MapperRegistry的getMapper方法：

```java
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null)
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

代码很简单就是传入sqlSession通过代理工厂创建mapper的实例，而工厂方法中通过动态代理技术产生实现类

```java
  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
```

所以关键是产生的动态代理类中的invoke方法：

```java
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (Object.class.equals(method.getDeclaringClass())) {
      try {
        return method.invoke(this, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```

到了这里我们可以看到最终执行的是mapperMethod的execute方法

```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    if (SqlCommandType.INSERT == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.insert(command.getName(), param));
    } else if (SqlCommandType.UPDATE == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
    } else if (SqlCommandType.DELETE == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
    } else if (SqlCommandType.SELECT == command.getType()) {
      if (method.returnsVoid() && method.hasResultHandler()) {
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        result = executeForMap(sqlSession, args);
      } else {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = sqlSession.selectOne(command.getName(), param);
      }
    } else {
      throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```

而excute方法使用的就是sqlSession的api，到此我们就完成了从mapper到sqlSession的绑定。总结起来就是通过动态代理的方式使得mapper与sqlSession绑定，最终执行的还是sqlSession中的方法。sqlSession也是一个接口，那他的最终实现类又在哪里呢，我们可以继续深入sqlSession中往下看。

sqlSession又sqlSessionFactory创建最终指向的创建方法是：

```java
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

一开始创建sql执行环境以及事务相关的对象，最终返回DefaultSqlSession对象，这个最后创建了DEfaultSqlSession对象，该对象主要有四个属性：

```java
  public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
    this.configuration = configuration;
    this.executor = executor;
    this.dirty = false;
    this.autoCommit = autoCommit;
  }
```



Configuration对象主要是数据库运行环境相关的信息，比如用户名密码，事务相关等等，而executor就是sql的执行器了。

在创建sqlSession需要传入执行器类型，执行器类型由一个枚举指定我们可以看到总共有三种

```java
public enum ExecutorType {
  SIMPLE, REUSE, BATCH
}
```

稍后讨论不同类型的执行器的区别。

我们在执行第一个插入用户的操作时首先进入动态代理的方法，通过代码跟进发现最终在SimpleExecutor中执行，其中SimpleExcutor，ResuseExcutor，BatchExcutor都继承自抽象类BaseExcutor。

```java
  public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.update(stmt);
    } finally {
      closeStatement(stmt);
    }
  }
```

在这个SimpleExcutor执行器中我们可以看到很多熟悉的东西比如Statement，对就是jdbc中的statement，最终执行的就是对jdbc中对sql的预编译对象，自此就完成了mybatis的一个操作流程。