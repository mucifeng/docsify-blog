### FactoryBean 和 BeanFactory区别
    以Factory结尾，表示他是一个工厂类接口，用于管理Bean的一个工厂，在spring中BeanFactory是ioc容器的核心接口，
      它的职责包括：实例化，定位，配置应用程序中的对象及建立这些对象间的依赖。
    FactoryBean：
      以Bean结尾，表示它是一个Bean。不用于其他Bean的是：它实现了FactoryBean接口的Bean，根据该Bean的ID从BeanFactory
      中获取的实际上是FactoryBean的getObject()返回的对象，而不是FactoryBean本身，如果需要获取FactoryBean对象，请
      在ID前面加一个&符号来获取。
### 代码验证
```java
public class TestFactoryBean {
	public void test(){ System.out.println("TestFactoryBean"); }
}

@Component
public class BasicFacoryBean implements FactoryBean<TestFactoryBean> {
	@Override
	public TestFactoryBean getObject() throws Exception { return new TestFactoryBean(); }
	@Override
	public Class<?> getObjectType() { return TestFactoryBean.class; }
}
public static void main(String[] args){
    TestFactoryBean bean = applicationContext.getBean(TestFactoryBean.class);
    bean.test();
}
```
FactoryBean的作用是提供更加灵活的Bean类型，假设TestFactoryBean是接口类型，没有任何实现。但是应用确实也需要用到接口对象（Mybatis中只有interface没有具体实现类）。
当需要容器存入对应的Bean时，就可以使用FactoryBean，在getObject方法里创建关于接口的代理类并返回。
```java
public interface TestInterfaceFactoryBean {
	void test();  //无任何实现类
}

@Component
public class BasicFacoryBean implements FactoryBean<TestInterfaceFactoryBean> {
	@Override
	public TestInterfaceFactoryBean getObject() throws Exception {
		InvocationHandler subjectProxy = new BeanProxy();
		TestInterfaceFactoryBean bean = (TestInterfaceFactoryBean) Proxy.newProxyInstance(subjectProxy.getClass().getClassLoader(),
				new Class<?>[] {TestInterfaceFactoryBean.class} ,
				subjectProxy); //JDK动态代理生成接口的代理类
		return bean;
	}
	@Override
	public Class<?> getObjectType() {
		return TestInterfaceFactoryBean.class;
	}
	class BeanProxy implements InvocationHandler {
		@Override
		public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
			System.out.println("--------------begin-------------");
			System.out.println("--------------end-------------");
			return "OK";
		}
	}
}
public static void main(String[] args){
    TestInterfaceFactoryBean bean = applicationContext.getBean(TestInterfaceFactoryBean.class);
    bean.test();
}
输出结果
--------------begin-------------
--------------end-------------
```
可以联想到Mybaits中，只定义interface,接口和XML文件关联就能找到方法所对应的XML节点，底层就是使用的FactoryBean技术中生成代理类。

### 源码探究

