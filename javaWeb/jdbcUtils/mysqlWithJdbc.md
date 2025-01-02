## 																		JDBC operate MySql 

### 准备阶段

1. 需要的包: 

   * ```commons-beanutils.jar```     提供数据库操作函数

   * ```commmons-collections.jar```
   * ```mysql-connector-java.jar```  数据库连接
   * ```druid-1.2.20.jar```  数据库连接池

2. 需要的配置文件```jdbc.properties```

   ```properties
   username=yourMysqlusername
   password=yourpassword
   url=jdbc:mysql://localhost:3306/yourDatase?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
   driverClassName=com.mysql.cj.jdbc.Driver
   initialSize=yourInitialNumber
   maxActive=yourMaxCount
   ```

### 基本操作

* 建立连接这里用连接池连接 通过数据库事务来管理可以用filter来检查错误

  ```java
  public class JdbcUtils{
      private static DuridDataSource druidDataSource;
      // 建立线程安全的连接 是整个项目共用一个连接 接保证了节约资源又可以保证数据库的操作安全
      private static ThreadLocal<Connection> connectionThreadLocal = new ThreadLocal();
  }
  ```

  * 初始化数据

  ```java
  static{
      try{
          Properties properties = new Properties();
          //读取jdbc.properties配置文件 currentClass 是你当前创建的类名
          InputStream resourceAsStream = currentClass.class.getClassLoader().getResourceAsStream("jdbc.prperties");
           //从流中加载数据
      	properties.load(resourceAsStream);
          // 创建数据库连接池
          druidDataSource = (DruidDataSource) DuridDataSourceFactory.createDataSource(properties);
     
      }catch(Exception e){
          throw new RuntimeException(e);
      }
  }
  ```

  * 建立连接

  ```java
  public static Connection getConnection()
  {
      Connection connection = connctionThreadLocal.get();
      
      if(connection == null)
      {
          try {
              connection = druidDataSource.getConnection();
              connectionThreadLocal.put(connection);
              connection.setAutoCommit(false);
              //设置手动设置事务
              // 这样可以保证数据库的安全避免添加垃圾数据
          } catch (SQLException e) {
               throw new RuntimeException(e);
          }
      }
  }
  ```

  * 提交事务

  ```java
  public static void submitAndClose()
  {
   	Connection connetion = connectionThreadLocal.get();
      if(connection != null)
      {
          try{
              connection.submit();
          }catch(SQLException e){
              e.printStackTrace();
          }finally{
              try{
                 connection.close(); 
              }catch(SQLException e){
                  e.printStackTrace();
              }
          }
      }
      connectionThreadLocal.remove();
      // 连接池底部实现用到了线程池如果不移除可能会报错
  }
  ```

  * 回滚事务

  ```java
  public static void rollBackAndClose()
  {
   	Connection connection = connectionThreadLocal.get();
      if(connection != null)
      {
          try{
              connection.rollback();
          }catch(SQLException e){
              e.printStackTrace();
          }finally{
               try{
                 connection.close(); 
              }catch(SQLException e){
                  e.printStackTrace();
              }  
          }
      }
      connectionThreadLocal.remove();
  }
  ```

* 添加|删除|修改 在BaseDao层

  ```java
  //import org.apache.commons.dbutils.QueryRunner;
     /**
       *
       * @param type 返回对象的类型
       * @param sql 执行的SQL语句
       * @param args 表示sql对应的参数值
       * @return 返回一个JavaBean的sql语句
       * @param <T> 返回类型的泛式
       */
  public int update(String sql, Object... args)
  {
      Connection connection = JdbcUtils.getConnection();
     	try{
          return queryRunner.update(connection, sql, args);
      }catch(SQLException e){
          throw new RuntimeExcetion(e);
          // 抛给JdbcUtil类 处理
      }
  }
  ```

* 查找

  ```java
      public <T> T queryForOne(Class<T> type, String sql, Object... args){
           Connection connection = JdbcUtils.getConnection();
             try {
                 //new BeanHandler<>(type) 将查询的数据用类处理器装换成T类
                  return queryRunner.query(connection, sql, new BeanHandler<>(type), args);
              } catch (SQLException e) {
                 throw new RuntimeException(e);
              }
      }
  
  //查询多个
     /**
       * new BeanHandler<>(type) 用于将查询结果映射为 Java 对象。
       * type 参数指定了目标 Java 类型.
       * 这段代码执行一个数据库查询并返回查询结果作为指定类型的 Java 对象
       * @return 查询返回多个JavaBean的sql语句
       */	
      public <T> List<T> queryForList(Class<T> type, String sql, Object ... args){
          Connection connection = JdbcUtils.getConnection();
          try {
              return queryRunner.query(connection, sql, new BeanListHandler<>(type), args);
          } catch (SQLException e) {
              throw new RuntimeException(e);
          }
      }
  
  	/**
       * @param sql 执行sql数据
       * @param args sql数据对应的参数
       * @return 返回一行一列的sql数据
       */
  //一般用来查询数量之类的
      public Object queryForSingleValue(String sql, Object ... args) {
          Connection connection = JdbcUtils.getConnection();
          try {
              return queryRunner.query(connection, sql, new ScalarHandler<>(), args);
          }catch (SQLException e) {
             throw new RuntimeException(e);
          }
      }
  ```

  

