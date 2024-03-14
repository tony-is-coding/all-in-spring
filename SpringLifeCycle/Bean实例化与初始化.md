## 阶段一、Bean实例化过程
### 实例化前置
Spring 在Bean实例化前提供了一个 实例化前置 AwarePostProcessor, 允许提前返回一个手动创建的bean, 这样可以绕过 Spring 容器的创建而手动创建对应的Bean;  这个步骤发生在Bean实例化前,  具体的代码位置在`AbstractAutowireCapableBeanFactory#createBean` 对 `AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation`的调用
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661424434508-72e59500-04cd-4508-9a6c-84c8a1168434.png#averageHue=%232c2c2c&clientId=u9b97e286-4b34-4&from=paste&height=268&id=u3ce2adcb&originHeight=268&originWidth=960&originalType=binary&ratio=1&rotation=0&showTitle=false&size=44236&status=done&style=none&taskId=u211b1d61-bb92-4459-b5a7-b5249770a65&title=&width=960)
该方法内部会先执行实例化前置拦截BeanPostProcessor方法: `InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation`, 如果有一个前置处理返回了一个不为空的对象, 那么就会终止前置处理, 并跳过后续的Bean实例化过程,  直接进入到  **初始化后后置**处理阶段`AutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization`

可以自己实现一个`InstantiationAwareBeanPostProcessor`来对所有用户Bean进行拦截处理
```java
@Component
public class MyInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {
	@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		// 进行实例化前置拦截, 可以在这个阶段返回一个初始化好的Bean
		if(ConditionOnCase){
			return new AlreadyBean();
		}
		return InstantiationAwareBeanPostProcessor.super.postProcessBeforeInstantiation(beanClass, beanName);
	}
}
```
### 实例化中
整个实例化中再Spring中体现在: `AbstractAutowireCapableBeanFactory#createBeanInstance`
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661477701187-ae9a112c-6b16-4019-a56b-7ad1ee1c0d38.png#averageHue=%232c2c2b&clientId=u9b97e286-4b34-4&from=paste&height=362&id=udda59a5b&originHeight=362&originWidth=956&originalType=binary&ratio=1&rotation=0&showTitle=false&size=68438&status=done&style=none&taskId=uc978f3c4-122e-4bf0-93e2-365ae379451&title=&width=956)
这个方法是整个SpringBean 对象实例化的过程体现, 包括以下核心的体现

- 构造方法推断过程
- **构造方法有参数情况下 自动注入的实现** : 在方法`AbstractAutowireCapableBeanFactory#autowireConstructor`, 基于Bean工厂注入策略有不同

针对构造方法的推断Spring也提供了对应的拓展: `SmartInstantiationAwareBeanPostProcessor` , 能够拦截构造方法推断过程, 自定义构造方法选择逻辑
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661478151146-00ad5281-3e22-4c0f-b706-062bec2d03c7.png#averageHue=%232d2c2b&clientId=u9b97e286-4b34-4&from=paste&height=329&id=u1434f767&originHeight=329&originWidth=840&originalType=binary&ratio=1&rotation=0&showTitle=false&size=44055&status=done&style=none&taskId=u008c73f6-5156-4528-af50-ef0f5196112&title=&width=840)
以下两种情况输出结果是完全不同的
```java
// 参照Bean
public class BeanInstanceBean {

	private BeanInstanceConstructInjectBean beanInstanceConstructInjectBean;
	public BeanInstanceBean(BeanInstanceConstructInjectBean beanInstanceConstructInjectBean) {
		this.beanInstanceConstructInjectBean = beanInstanceConstructInjectBean;
	}
	public BeanInstanceBean() {}
}

// 构造推导插件
public class MySmartInstantiationAwareBeanPostProcessor implements SmartInstantiationAwareBeanPostProcessor {

	@Override
	public Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName) throws BeansException {
		if(BeanInstanceBean.class.equals(beanClass)){
			Constructor<?>[] constructors = beanClass.getDeclaredConstructors();
			for(Constructor<?> c:constructors){
				if(c.getParameterTypes().length !=0){ // 返回一个参数的
					return new Constructor[]{c};
				}
			}
		}
		return SmartInstantiationAwareBeanPostProcessor.super.determineCandidateConstructors(beanClass, beanName);
	}
}

@Test
public void testMySmartInstantiationAwareBeanPostProcessor(){
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(BeanInstanceConfiguration.class);
    ctx.register(BeanInstanceBean.class);
    BeanInstanceBean bean = ctx.getBean(BeanInstanceBean.class);
    System.out.println(bean);
}    
//output1: 
// BeanInstanceBean{beanInstanceConstructInjectBean=null}

@Test
public void testMySmartInstantiationAwareBeanPostProcessor(){
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(BeanInstanceConfiguration.class);
    ctx.getBeanFactory().addBeanPostProcessor(new MySmartInstantiationAwareBeanPostProcessor());
    ctx.register(BeanInstanceBean.class);
    BeanInstanceBean bean = ctx.getBean(BeanInstanceBean.class);
    System.out.println(bean);
}
//output2: 	BeanInstanceBean{beanInstanceConstructInjectBean=org.cnc.explain.liftcycle.beaninstance_6.BeanInstanceConstructInjectBean@2c1b194a} 

```

此外，还提供了BeanDefinition级别的两个入口,能够拦截到实例化,提供一个bean实例返回, **使用场景逐渐被放弃**

- Instance Supplier
- factory-method
### 实例化后置
Spring在Bean对象的实例化完成后, 还会进行**三个**核心的操作
#### 1、 Bean缓存提前曝光
将Bean对象加入到三级缓存`**singletonFactories**`中去,  这个时候将创建中的对象提前曝光到工厂集合中, 后续有同样的方法来执行`doGetBean`操作会提前走到三级缓存,  就可以在互斥的锁情况下进行对象的同步创建了
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661694988236-81e59475-f436-44fd-aa6e-acda1a942f02.png#averageHue=%23353332&clientId=u352ea755-d71b-4&from=paste&height=736&id=u49b507b9&originHeight=736&originWidth=957&originalType=binary&ratio=1&rotation=0&showTitle=false&size=141655&status=done&style=none&taskId=u2f4a61df-1990-4962-bf57-30f4633e407&title=&width=957)
#### 2、 `**MergedBeanDefinitionPostProcessor**`
进行实例化的后置的 `**MergedBeanDefinitionPostProcessor#postProcessorMergedBeanDefinition**`处理
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661821908238-5275b0c3-0aaf-41b3-8bea-3c36506c1577.png#averageHue=%23333232&clientId=u58396178-5560-4&from=paste&height=292&id=u2997dda0&originHeight=292&originWidth=802&originalType=binary&ratio=1&rotation=0&showTitle=false&size=38246&status=done&style=none&taskId=uf9aee80c-725e-4725-a631-b8882a2c599&title=&width=802)
默认情况下,  Spring只添加了`ApplicationListenerDetector` 这一个Processor, 这个processor 用于收集那些实现了 `ApplicationListener`的Bean, 算是对Spring事件驱动模型的一种支持;  
这个阶段可以做的事情比较多, 但是比较突出的机制是:  对已经完成父子合并的BeanDefinition进行一次后置处理,  后续的多个步骤如属性设置, 初始化等都会依赖这个MergedBeanDefinition;
**可以通过多个实现Processor收集一些元信息在Bean的生命周期各个阶段进行应用,  但是场景少见**
**针对这个用法，目前没有很好的应用场景， 待发掘中...**
#### 3、 实例化后置 postProcessorAfterInstantiation
在进入正在的属性设置前, Spring提供了一个实例化后置处理入口: `InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation`,  这个方法执行的地点在:  `AbstractAutowireCapableBeanFactory#populateBean`, 主要的作用有

- 提前对字段进行注入或者修改等
```java
public class MyAfterInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        if("beanInstanceBean".equals(beanName)){
            BeanInstanceBean ob = (BeanInstanceBean) bean;
            ob.setBeanInstanceConstructInjectBean(
                new BeanInstanceConstructInjectBean("changed by postProcessAfterInstantiation")
            );
        }
        return InstantiationAwareBeanPostProcessor.super.postProcessAfterInstantiation(bean, beanName);
    }
}
// 输出
BeanInstanceBean{
    beanInstanceConstructInjectBeanAutowired=BeanInstanceConstructInjectBean{name='null'}
beanInstanceConstructInjectBean=BeanInstanceConstructInjectBean{name='changed by postProcessAfterInstantiation'}
    }
```

- 通过返回false提前结束字段注入工作
```java
public class MyAfterInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {
    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        if("beanInstanceBean".equals(beanName)){
            return false;
        }
        return InstantiationAwareBeanPostProcessor.super.postProcessAfterInstantiation(bean, beanName);
    }
}
//输出
BeanInstanceBean{
    beanInstanceConstructInjectBeanAutowired=null
    beanInstanceConstructInjectBean=null
    }
```
> 因为这个阶段的拦截得倒的是一个未进行属性注入的Bean对象,  对Bean的改造会有所限制


## 阶段二、**Bean属性设置阶段**
Spring Bean的属性设置阶段核心发生在`AbstractAutowireCapableBeanFactory#populateBean`
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661779396433-e61564c7-5579-4c27-a6c9-f5ce255b3402.png#averageHue=%232d2d2c&clientId=u58396178-5560-4&from=paste&height=161&id=u4979e5df&originHeight=161&originWidth=830&originalType=binary&ratio=1&rotation=0&showTitle=false&size=31138&status=done&style=none&taskId=u04c43081-7395-48c3-beeb-78583bb88e8&title=&width=830)
这个阶段主要的工作就是:  对Bean的属性进行设置,  **同样就包括了对于Bean属性的依赖注入,  我们常知的@Autowired  和 @Resource 就是这个阶段进行注入的;**
概括来讲, 这个阶段可以分为两个步骤

1. 通过 @Autowird 或者 @Resource 注入
2.  属性值的应用
###  @Autowird 或者 @Resource 注入
整个的代码执行在 `AbstractAutowireCapableBeanFactory#populateBean`方法内,  这个阶段通过一个 `InstantiationAwareBeanPostProcessor`进行一次拦截处理, 完成属性的注入
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661910202955-bfe3b8ba-280d-4610-882b-47cf585d4599.png#averageHue=%232c2c2c&clientId=u5daf9a4a-7269-4&from=paste&height=382&id=u75e20d3c&originHeight=382&originWidth=961&originalType=binary&ratio=1&rotation=0&showTitle=false&size=69776&status=done&style=none&taskId=u59e77dc7-dc92-480e-95c7-564c76dcd90&title=&width=961)
默认情况下, Spring 会在初始化容器阶段注入三个标准的 `InstantiationAwareBeanPostProcessor`:
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661910322307-f239a797-a6f5-4d3e-8163-91a977c5e9ed.png#averageHue=%233e4143&clientId=u5daf9a4a-7269-4&from=paste&height=90&id=uc146d640&originHeight=90&originWidth=585&originalType=binary&ratio=1&rotation=0&showTitle=false&size=17612&status=done&style=none&taskId=u7d12b6a2-576e-438a-b8de-131fbda729f&title=&width=585)
这个阶段比较重要的是后面的两个

- **CommonAnnotationBeanPostProcesor**

这个类主要是Spring 实现用来支持 JSR-250 标准的;
在这个阶段这个Processor 核心处理的是 `@Resource` 注解的注入

- **AutowirdAnnotationBeanPostProcessor**

这个类Spring 主要是用来提供Spring标准的注入注解处理以及 JSR-330的部分注解支持;
在这个阶段这个Processor 核心处理的是 `@Autowired` 注解的注入, 当然这个注解还提供了对 `@Value`  和 `@Inject` 的支持

同时可以直接实现一个 `InstantiationAwareBeanPostProcessor` 对属性设置前置的处理, 进行属性拦截自定义等;
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661911071796-17cafdb6-0385-4d18-ba72-c208ef6ea638.png#averageHue=%23332c2b&clientId=u5daf9a4a-7269-4&from=paste&height=385&id=u114be4c4&originHeight=385&originWidth=1054&originalType=binary&ratio=1&rotation=0&showTitle=false&size=74218&status=done&style=none&taskId=u86864240-2f5a-449d-a136-6c152aced44&title=&width=1054)
特别需要注意的是, 这个阶段分别调用了两个钩子函数 :

- `**postProcessProperties**`
- `**postProcessPropertyValues **`** **

其中 `postProcessPropertyValues` 已经被官方标记为弃用了,  内部很多实现都已经转向调用 `postProcessorProperties` 了, 不建议使用这种方式
因为 使用`postProcessPropertyValues` 返回一个`null` 就会导致属性设置提前结束,进入到初始化缓解, 在这里看起来毫无意义

### 属性值的应用
这里就是属性的应用了, 先将属性依赖注入值解析成 PropertyValues对象, 然后通过循环的调用反射set 对属性值进行设定
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661950092832-9b4e93c6-cde7-4da5-9cb8-e1f80c00139d.png#averageHue=%232f2e2e&clientId=u43c14520-3218-4&from=paste&height=97&id=u6d8dcabf&originHeight=97&originWidth=811&originalType=binary&ratio=1&rotation=0&showTitle=false&size=15832&status=done&style=none&taskId=ud4345404-7736-4842-a6b1-f23d1a14dd8&title=&width=811)

## 阶段三、**Bean初始化阶段**
Spring 的初始化整个可以理解为围绕着执行初始化方法这个动作, 以及这个动作前后发生的一系列调用, 整个方法的执行在 `AbstractAutowireCapableBeanFactory#initializeBean`
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661952852124-e7a7ed91-6684-4182-a66b-c221e03cfb17.png#averageHue=%232c2c2b&clientId=u43c14520-3218-4&from=paste&height=280&id=u8d594a95&originHeight=280&originWidth=881&originalType=binary&ratio=1&rotation=0&showTitle=false&size=53910&status=done&style=none&taskId=u2ad78590-f759-482e-8e12-03226c598fb&title=&width=881)
### BeanAware回调阶段
初始化对BeanAware相关方法的回调发生在 `AbstractAutowireCapableBeanFactory#invokeAwareMethods`, 在这个阶段会依次对三个Aware接口进行回调

1. BeanNameAware
2. BeanClassLoaderAware
3. BeanClassLoaderAware

简单示例下:
```java
public class BeanAwareBean implements BeanNameAware, BeanFactoryAware, BeanClassLoaderAware {
	@Override
	public void setBeanClassLoader(ClassLoader classLoader) {
		System.out.println("call BeanClassLoaderAware method");
	}
	@Override
	public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
		System.out.println("call BeanFactoryAware method");
	}
	@Override
	public void setBeanName(String name) {
		System.out.println("call BeanNameAware method");
	}
}


@Test
public void testBeanAwareBeanCallbacks() {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(BeanInitConfiguration.class);
    ctx.register(BeanAwareBean.class);
    System.out.println("完成bean注入");
    BeanAwareBean bean = ctx.getBean(BeanAwareBean.class);
    System.out.println(bean);
}
// 输出
完成bean注入
call BeanNameAware method
call BeanClassLoaderAware method
call BeanFactoryAware method
org.cnc.explain.lifecycle.beaninitial_8.BeanAwareBean@31fa1761
```
> 有一个小细节:   通过factory/application 手动注册进去的Bean只是注册了Beandefinition, 真正发生Bean初始化是在`getBean`时, 通过XML或Annotation加载的则不一样, 因为在Applicaiton `refresh`阶段则会加载所有的`**非懒加载Bean**`
> ![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661995996159-1adb86f4-78f0-4845-9441-6c43c70087db.png#averageHue=%23535c2d&clientId=u868d4ab0-a939-4&from=paste&height=78&id=u3e77b938&originHeight=78&originWidth=975&originalType=binary&ratio=1&rotation=0&showTitle=false&size=23298&status=done&style=none&taskId=ud32f7568-6100-4207-bfee-802dbbdd561&title=&width=975)


对Aware 相关方法支持在这个阶段做一些自定义注入或者缓存之类的
### Bean初始化前置
初始化前置就是在初始化方法执行前的一系列回调,  主要是调用`BeanPostProcessor#postProcessorBeforeInitialization`这个回调, 默认情况下会包含6个基本的Processor
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661997233181-55e44e2f-97f8-45e3-bf07-9af60dba89c8.png#averageHue=%23373a3c&clientId=u868d4ab0-a939-4&from=paste&height=129&id=u8d15bb8b&originHeight=129&originWidth=609&originalType=binary&ratio=1&rotation=0&showTitle=false&size=28324&status=done&style=none&taskId=u6a38b646-0fdf-4fc3-b235-009e770b69a&title=&width=609)
但是实际上,只有 `ApplicationContextAwareProcessor` 在这个的实现有实际的作用,  具体代码可以看`ApplicationContextAwareProcessor#postProcessBeforeInitialization`
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1662038320857-35cab3ca-74ff-4d35-a0f9-37157623c8cb.png#averageHue=%23272525&clientId=ue77b7ba8-4cd9-4&from=paste&height=627&id=u47d1aeb0&originHeight=627&originWidth=963&originalType=binary&ratio=1&rotation=0&showTitle=false&size=87867&status=done&style=none&taskId=ud910147c-89d1-440c-9813-87f8f553894&title=&width=963)
这里有个重点是会执行一些 `Aware` 的接口

- EnvironmentAware
- EmbeddedValueResolverAware
- ResourceLoaderAware
- ApplicationEventPublisherAware
- MessageSourceAware
- ApplicationStartupAware
- ApplicationContextAware

比较常用的就是 `ApplicationContextAware`一般会用来作为一个通用的 Bean 管理工具, 提供一个ApplicationContext 的getBean 入口
```java
class BeanHolder implements ApplicationContextAware{
    private ApplicationContext applicationContext;
    public void setApplicationContext(ApplicationContext ctx){
        this.applicationContext = ctx;
    }
    public T getBean(Class<T> type){
        return applicationContext.getBean();
    }
}
```

### Bean初始化
初始化过程就是执行选择的执行初始化方法, Spring 支持的指定Bean的初始化方法的三种方法

1. 基于XML配置指定 `init-method`
2. 基于注解 @Bean(initMethod="xx")
3. 通过手动注入初始化方法属性到  `AbstractBeanDefinition#initMethodName`

这三种方法从根本上来说是互相独立的,   因为配置Bean的方式是不同的入口

Spring 执行 Bean初始化的整个过程发生在 `AbstractAutowireCapableBeanFactory#invokeInitMethods`
需要特别注意的, 在执行真正的初始化方法之前, 会去执行一个属性设置完成后置调用: `InitializingBean#afterPropertiesSet`,  这个回调提供了一次初始化执行前对Bean的属性进行修改的机会,   **但是需要注意这个方法的执行的阶段处在初始化方法执行之前**
可以做一个简单的测试
```java
public class PriorityAfterPropertiesSetAndInitMethodBean implements InitializingBean {
	private String name;

	@Override
	public void afterPropertiesSet() throws Exception {
		this.name = "from propertiesSet";
		System.out.println("InitializingBean # afterPropertiesSet invoke");
	}
	public void initMethod(){
		this.name = "from init method";
		System.out.println("init method invoke");
	}

	@Override
	public String toString() {
		return "PriorityAfterPropertiesSetAndInitMethodBean{" +
				"name='" + name + '\'' +
				'}';
	}
}
@Bean(initMethod = "initMethod")
public PriorityAfterPropertiesSetAndInitMethodBean priorityAfterPropertiesSetAndInitMethodBean(){
    return new PriorityAfterPropertiesSetAndInitMethodBean();
}
@Test
public void testPriorityAfterPropertiesSetAndInitMethod(){
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(BeanInitConfiguration.class);
    PriorityAfterPropertiesSetAndInitMethodBean bean = ctx.getBean(PriorityAfterPropertiesSetAndInitMethodBean.class);
    System.out.println(bean);
}

//输出
InitializingBean # afterPropertiesSet invoke
init method invoke
```
之后就是执行具体的初始化方法`AbstractAutowireCapableBeanFactory#invokeCustomInitMethod`, 就是简单的通过反射获取对应的方法去执行
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1662042871217-9fb682e6-71ba-47a6-8655-4c7f54f242d3.png#averageHue=%232e2524&clientId=ue77b7ba8-4cd9-4&from=paste&height=190&id=u904cdfbf&originHeight=190&originWidth=819&originalType=binary&ratio=1&rotation=0&showTitle=false&size=47491&status=done&style=none&taskId=ued18b268-58b1-4ed2-ab81-23d42733799&title=&width=819)

### Bean初始化后置
Spring Bean的初始化后置主要是执行了一次后置回调: `BeanPostProcessor#postProcessAfterInitialization`
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1662043211245-aa3ce389-2bdf-4c2a-bf73-172b470b4cfe.png#averageHue=%23282828&clientId=ue77b7ba8-4cd9-4&from=paste&height=129&id=u4c57ef60&originHeight=129&originWidth=953&originalType=binary&ratio=1&rotation=0&showTitle=false&size=23945&status=done&style=none&taskId=ude6f2ec4-6101-4895-aa46-e244f22925d&title=&width=953)
可以看到,  这个回调方法返回的是一个bean对象, 所以可以在这个阶段对 bean进行定制化改造, 甚至是返回一个全新的bean
简单测试一下
```java
public class PostProcessAfterInitializationBean {
	private String name;
	public PostProcessAfterInitializationBean() {
		name = "default";
	}
	public void setName(String name) {
		this.name = name;
	}
	@Override
	public String toString() {
		return "PostProcessAfterInitializationBean{" +
				"name='" + name + '\'' +
				'}';
	}
}
-- 测试修改属性
@Test
public void testChangeBeanPropertiesByPostProcessAfterInitialization(){
	AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(BeanInitConfiguration.class);
	ctx.getBeanFactory().addBeanPostProcessor(
			new BeanPostProcessor() {
				@Override
				public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
					if("postProcessAfterInitializationBean".equals(beanName)){
						try {
							Method method = bean.getClass().getMethod("setName",String.class);
							method.invoke(bean,"from postProcessAfterInitializationBean");
						} catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {
							e.printStackTrace();
						}
					}
					return BeanPostProcessor.super.postProcessAfterInitialization(bean, beanName);
				}
			}
	);
	ctx.register(PostProcessAfterInitializationBean.class);
	PostProcessAfterInitializationBean bean = ctx.getBean(PostProcessAfterInitializationBean.class);
	System.out.println(bean);
}
// 输出
PostProcessAfterInitializationBean{name='from postProcessAfterInitializationBean'}

-- 测试返回一个全新的bean
@Test
public void testReturnNewBeanByPostProcessAfterInitialization(){
	AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(BeanInitConfiguration.class);
	ctx.getBeanFactory().addBeanPostProcessor(
			new BeanPostProcessor() {
				@Override
				public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
					if("postProcessAfterInitializationBean".equals(beanName)){
						PostProcessAfterInitializationBean newBean=  new PostProcessAfterInitializationBean();
						newBean.setName("from new Bean");
						return newBean;
					}
					return BeanPostProcessor.super.postProcessAfterInitialization(bean, beanName);
				}
			}
	);
	ctx.register(PostProcessAfterInitializationBean.class);
	PostProcessAfterInitializationBean bean = ctx.getBean(PostProcessAfterInitializationBean.class);
	System.out.println(bean);
}
// 输出
PostProcessAfterInitializationBean{name='from new Bean'}
```

## 阶段四、**所有单例bean初始化完成后阶段**
当所有的懒加载Bean都被加载完成后,Spring会遍历已经加载的Bean, 找出其中的  `SmartInitializingSingleton`,执行他们的 `afterSingletonsInstantiated`方法;   这个步骤的入口在 `DefaultListableBeanFactory#preInstantiateSingletons`
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1662044621649-d4c5fea4-78bb-45c3-9e62-a417ec085eb0.png#averageHue=%23262626&clientId=ue77b7ba8-4cd9-4&from=paste&height=401&id=ua21bab27&originHeight=401&originWidth=990&originalType=binary&ratio=1&rotation=0&showTitle=false&size=77476&status=done&style=none&taskId=u7552ec18-0d12-4e12-a0f0-13fe96c3be4&title=&width=990)
> 可以在这个阶段进行一些通知通告的调用,或者一些缓存的加载了,  因为到了这里基本上是确定所有的Bean都已经初始化完成


