```xml
<beans>
    <bean id="write_DB" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
    </bean>
    <bean id="read_DB" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
    </bean>
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="write_DB" />
    </bean>
    <tx:annotation-driven transaction-manager="transactionManager" />
    <bean id="routeDS" class="route.RouteDS">
        <!-- 写数据源，一主 -->
        <property name="writeDataSource" ref="write_DB" />
        <!-- 读数据源，多从 -->
        <property name="readDataSoures">
            <list>
                <ref bean="read_DB" />
            </list>
        </property>
    </bean>
</beans>
```
```java
@Test
@Transactional("transactionManager")
@Rollback
public void test___add() throws Exception {
    service.select();
    Object o = new Object();
    service.add(o);
}
```
## Mybatis读写分离配置中，先查后写导致异常发生
```java
//spring中的事务处理器 XML中配置的TransactionManager
public class DataSourceTransactionManager{
    	//开始事务
    	protected void doBegin(Object transaction, TransactionDefinition definition) {
    	    if (!txObject.hasConnectionHolder() || txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
    	         //获取一个connection
                Connection newCon = this.dataSource.getConnection();
                if (logger.isDebugEnabled()) {
                    logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
                }
                txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
            }
            if (txObject.isNewConnectionHolder()) {
                //事务方法写入ThreadLocal中
                //结构如下,方便后续获取Connection操作都再同一个Connection中
                TransactionSynchronizationManager.bindResource(getDataSource(), txObject.getConnectionHolder()); 
            }
    	}
}
public abstract class TransactionSynchronizationManager {
     //ConnectionHolder放入resources中
     //数据结构为  DataSource -> ConnectionHolder
     private static final ThreadLocal<Map<Object, Object>> resources = new NamedThreadLocal<Map<Object, Object>>("Transactional resources");
}
```
由上面代码可以看到，Spring中的线程实际上也是在DataSourceTransactionManager中根据dataSource创建Connection，并将相关信息放入ThreadLocal中。而在Mybatis中要使用此次创建的Connection.
引入mybatis-spring.jar包，里面有SpringManagedTransaction.java。当Mybatis代码运行到获取Connection时。实际是调用spring-jdbc.jar包中的工具类获取连接。
```java
public class SpringManagedTransaction｛
    public Connection getConnection() throws SQLException {
        if (this.connection == null) {
          openConnection();
        }
        return this.connection;
    }
  private void openConnection() throws SQLException {
    this.connection = DataSourceUtils.getConnection(this.dataSource); //调用spring-jdbc.jar包中的util类，获取连接
    this.autoCommit = this.connection.getAutoCommit();
    this.isConnectionTransactional = DataSourceUtils.isConnectionTransactional(this.connection, this.dataSource);
  }
}
public class DataSourceUtils{
    public static Connection doGetConnection(DataSource dataSource) throws SQLException {
        //通过TransactionSynchronizationManager获取连接
        //底层方法是使用当前线程获得 Map，再传入dataSource获取最终的ConnectionHolder
        ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource); 
    }       
}
```
### 问题解析
	第一步：首先使用@Transactional("transactionManager")后，在代码第一次运行到doBegin()时，TransactionSynchronizationManager中的resources是Map数据结构即
    resources (write_DB -> write_ConnectionHolder)
    第二步：在运行到Test方法时，里面第一步是查询，Mybatis读写分离的插件会判断此次是“读”，则在后续传递的dataSource中使用的是  read_DB
    在方法运行到DataSourceUtils.doGetConnection(dataSource)时，此时是拿不到read_DB 对应的 ConnectionHolder，则此次会新生成  read_ConnectionHolder。
    resources (write_DB -> write_ConnectionHolder, read_DB -> read_ConnectionHolder)
    第三步：将生成的connection写入 SpringManagedTransaction.connection。后续的 service.add(o)方法会继续进入 SpringManagedTransaction.getConnection()方法中
    但是这次进入该方法时，会发现this.connection != null  则为Mybatis返回了 read_connection
### 问题后果
1. 如果读库不支持“写”功能，那么程序会异常，并且多数先“读”后“写”的方法体都会发生异常
2. 如果读库支持“写”功能，程序不会有任何异常发生，但是在数理逻辑上已经不是读写分离了，更像是分布式读写库。
在配置的事务基础上，由于第一步缓存的write_ConnectionHolder在后期没有用上，则在begin 和 rollback、commit之间没有SQL执行。

### 问题解决
1. 修改Mybatis读写分离拦截器。在观察到 TransactionSynchronizationManager.getResource(dataSource) 已经有了write_ConnectionHolder，表示对应的方法体是使用
@Transactional标识了的方法，将此次请求的DataSource全部指定为 write_DB
2. 减小Transactional的粒度，使用代码的方式为事务保驾护航，对于查询方法不放入代码块中
    