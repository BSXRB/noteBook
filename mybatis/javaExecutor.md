# javaExecutor

前文我们已经介绍了mybatis的工作流程，了解了真正执行sql的是Executor类，当然Executor除了执行sql之外还有管理缓存的管理。

为了了解Executor我们先看看他的继承结构。

![image-20200902122459602](D:\Note\img\Executor.png)

可以看到Executor有一个BaseExecutor的抽象类，这个抽象类定义了一些更新，查找以及缓存相关的基础方法，而这个抽象类有三个子类，就是mybatis定义的三种类型的执行器。CachingExecutor是二级缓存相关的类，二级缓存也是实现Executor接口的类，他有一个Executor类型的delegate属性，当处理完二级缓存相关的逻辑之后就交由delegate去处理，不管你的实现类是什么。这就实现了通过装饰器模式增加二级缓存功能。我们主要看Executor的三个实现类。我们既然知道了mybatis最终是由Executor执行的sql语句，那么我们就自己构建Executor来执行。

```java
public class ExecutorTest {
    private static Configuration configuration;
    private static Connection connection;
    private static JdbcTransaction transaction;
    public static void main(String[] args) throws SQLException {
        SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
        SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(User.class.getResourceAsStream("/mybatis-config.xml"));
        configuration = sqlSessionFactory.getConfiguration();
        connection = DriverManager.getConnection("jdbc:mysql://127.0.0.1:3306/mybatisread?serverTimezone=UTC","root","root");
        transaction = new JdbcTransaction(connection);
        MappedStatement mappedStatement = configuration.getMappedStatement("TestMapper.queryUser");
        SimpleExecutor executor = new SimpleExecutor(configuration,transaction);
        List<Object> list = executor.doQuery(mappedStatement, 1, RowBounds.DEFAULT, SimpleExecutor.NO_RESULT_HANDLER, mappedStatement.getBoundSql(1));
        System.out.println(list.get(0));
    }
}
```



我们先构建一个简单的SimpleExecutor执行器执行查询操作。

