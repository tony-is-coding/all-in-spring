# 一镜到底-全览Spring 生命周期
* [第一部分、生命周期阶段导图](#第一部分生命周期阶段导图)
* [第二部分、生命周期阶段详解](#第二部分生命周期阶段详解)
  * [阶段一、Bean 元信息配置与解析](#阶段一bean-元信息配置与解析)
    * [SpringBean 元对象之 BeanDefinition](#springbean-元对象之-beandefinition)
    * [Bean元信息配置与加载 3 种方式](#bean元信息配置与加载-3-种方式)
      * [1. 基于XML](#1-基于xml)
      * [2. 基于注解](#2-基于注解)
      * [3. 基于API](#3-基于api)
  * [阶段二、**Spring Bean注册阶段**](#阶段二spring-bean注册阶段)
  * [阶段三、**BeanDefinition合并阶段**](#阶段三beandefinition合并阶段)
  * [阶段四、**Bean Class加载阶段**](#阶段四bean-class加载阶段)
  * [阶段五、Bean实例化过程](#阶段五bean实例化过程)
    * [实例化前置](#实例化前置)
    * [实例化中](#实例化中)
    * [实例化后置](#实例化后置)
      * [1、 Bean缓存提前曝光](#1-bean缓存提前曝光)
      * [2、 `**MergedBeanDefinitionPostProcessor**`](#2-mergedbeandefinitionpostprocessor)
      * [3、 实例化后置 postProcessorAfterInstantiation](#3-实例化后置-postprocessorafterinstantiation)
  * [阶段六、**Bean属性设置阶段**](#阶段六bean属性设置阶段)
    * [@Autowird 或者 @Resource 注入](#autowird-或者-resource-注入)
    * [属性值的应用](#属性值的应用)
  * [阶段七、**Bean初始化阶段**](#阶段七bean初始化阶段)
    * [BeanAware回调阶段](#beanaware回调阶段)
    * [Bean初始化前置](#bean初始化前置)
    * [Bean初始化](#bean初始化)
    * [Bean初始化后置](#bean初始化后置)
  * [阶段八、**所有单例bean初始化完成后阶段**](#阶段八所有单例bean初始化完成后阶段)
  * [阶段九、**Bean销毁阶段**](#阶段九bean销毁阶段)
    * [1. AbstractAutowireCapableBeanFactory#destroyBean](#1-abstractautowirecapablebeanfactorydestroybean)
    * [2. ConfigurableBeanFactory#destroySingletons](#2-configurablebeanfactorydestroysingletons)
    * [DisposableBeanAdapter#destroy 调用的逻辑说明](#disposablebeanadapterdestroy-调用的逻辑说明)
      * [第一步、调用DestructionAwareBeanPostProcessor的postProcessBeforeDestruction](#第一步调用destructionawarebeanpostprocessor的postprocessbeforedestruction)
      * [第二步、 执行DisposableBean的destory](#第二步-执行disposablebean的destory)
      * [第三步、执行自定义的销毁方法](#第三步执行自定义的销毁方法)
* [第三部分、生命周期钩子](#第三部分生命周期钩子)


# 第一部分、生命周期阶段导图
![Bean 生命周期阶段 - 导图.jpg](https://cdn.nlark.com/yuque/0/2022/jpeg/22746802/1663118287776-9d2dc7b8-5b74-4007-a33d-373c1517e54e.jpeg#averageHue=%23f8f7f7&clientId=u61c26ef5-f7e8-4&from=ui&id=u630bed80&originHeight=2686&originWidth=2676&originalType=binary&ratio=1&rotation=0&showTitle=false&size=802616&status=done&style=none&taskId=ueb5412c9-4075-445c-8ed6-fbca2cae660&title=)

# 第二部分、生命周期阶段详解
## 阶段一、Bean 元信息配置与解析
> **大前提** : Spring Bean 在实例化初始化之前, 其实都是以元对象 - `BeanDefinition` 存在；Spring容器启动的过程中，会将Bean解析成Spring内部的BeanDefinition结构。不管是是通过xml配置文件的<Bean>标签，还是通过注解配置的@Bean，还是@Compontent标注的类，还是扫描得到的类，它最终都会被解析成一个 `BeanDefinition `对象，最后Spring Bean工厂就会根据这份Bean的定义信息，对bean进行实例化、初始化等操作;

### SpringBean 元对象之 BeanDefinition
> BeanDefinition描述了关于Bean定义的各种信息,  例如:
> - bean对应的class
> - scope
> - lazy信息
> - dependOn信息
> - autowireCandidate(是否是候选对象)
> - primary(是否是主要的候选者)等信息
> - 是否单例/多例



BeanDefinition是个接口，有几个实现类，看一下类图：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1659880643282-9849b710-de7e-46fc-8be8-9043e419cd9b.png#averageHue=%232e2e2e&clientId=u512fd58a-095c-4&from=paste&height=296&id=KUpv8&originHeight=592&originWidth=1720&originalType=binary&ratio=1&rotation=0&showTitle=false&size=50488&status=done&style=none&taskId=udd038046-7018-4d60-b318-c4a43f060c4&title=&width=860)
**BeanDefinition 核心分类**

- RootBeanDefinition:   表示根Bean定义的信息, 通常没有父Bean情况下直接使用这个表示
- ChildBeanDefinition:  表示子Bean的定义信息
- GenericBeanDefinition: 通用的Bean定义信息
- ConfigurationClassBeanDefinition:  通过配置类下 `@Bean`  方法定义的Bean的信息
- AnnotationClassBeanDefinition:  通过注解的方式定义

**BeanDefinition 继承了两个接口**

- AttributeAccessor

相当于一种 k-v 的数据结构, 底层是一种LinkedHashMap实现, 用来存储BeanDefinition过程中一些附属信息

- BeanMetadataElement

该接口提供一个  `getResource` 方法, 用于查找BeanDefinition定义的来源
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1659881128234-c8aa1f65-f8e5-47fd-bcdc-04b696291b2f.png#averageHue=%232e2e2e&clientId=u512fd58a-095c-4&from=paste&height=221&id=rMOX1&originHeight=442&originWidth=1724&originalType=binary&ratio=1&rotation=0&showTitle=false&size=26702&status=done&style=none&taskId=u7591bc23-2738-4034-92e6-e7bfb758063&title=&width=862)
### Bean元信息配置与加载 3 种方式
#### 1. 基于XML
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661411618384-488da484-e7b3-4498-8fb1-0691f32c6a77.png#averageHue=%232e2e2d&clientId=u9b97e286-4b34-4&from=paste&height=224&id=u98ff95b9&originHeight=224&originWidth=819&originalType=binary&ratio=1&rotation=0&showTitle=false&size=42518&status=done&style=none&taskId=u3d345295-b52a-495c-9030-de8e417b859&title=&width=819)
基于配置的容器启动过程如下,  本质上起来的是一个  `FileSystemXmlApplicationContext`对象
```java
public void testConfigBeanFromXml() {
    FileSystemXmlApplicationContext ctx = new FileSystemXmlApplicationContext("classpath:step1.xml");
    MetaConfigXmlBean bean = ctx.getBean(MetaConfigXmlBean.class);
    System.out.println(bean);
}
```
加载则是通过 `XmlBeanDefinitionReader`  进行相关文件的解析读取
```java
public void testParseFromXmlFile() {
    DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
    XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
    int count = reader.loadBeanDefinitions(new ClassPathResource("step1.xml"));
    System.out.println("total load " + count + " bean definitions");

    for (String definitionName : factory.getBeanDefinitionNames()) {
        BeanDefinition definition = factory.getBeanDefinition(definitionName);
        System.out.println(definition);
    }
}
// 通过XML配置解析出来的BeanDefinition都是 `GenericBeanDefinition` 类型
// 输出
Generic bean: class [org.cnc.explain.liftcycle.beanparse_2.ParsedXmlBean]; scope=; abstract=false; lazyInit=false; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=null; factoryMethodName=null; initMethodName=null; destroyMethodName=null; defined in class path resource [step2.xml]

Generic bean: class [org.cnc.explain.liftcycle.beanparse_2.ParsedXmlBean]; scope=; abstract=false; lazyInit=false; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=null; factoryMethodName=null; initMethodName=null; destroyMethodName=null; defined in class path resource [step2.xml]

```

#### 2. 基于注解
基于注解的容器启动由 `AnnotationConfigApplciationContext`完成
```java
@Component/@Service/@Repositry 
public class TestBean{
}
// 方式二、通过@Configuration 标准的配置加载
@Configuration
public class AppConfiguration{
    @Bean
    public TestBean  testBean(){return new TestBean();}
}
//方式一、bean类上加对应的注解 这种需要采用注解 @ComponentScan扫描
@Test 
public void testConfigurationFromAnnotation(){ 
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplciationContext(AppConfiguraion.class);
    TestBean tb = ctx.getBean(TestBean.class);
}

```
解析交由给 `AnnotatedBeanDefinitionReader`完成
```java

public void testParseFromAnnotationConfiguration() {
    DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
    AnnotatedBeanDefinitionReader reader = new AnnotatedBeanDefinitionReader(factory);
    // 注册到 bean-factory
    reader.register(MetaConfigBean.class);
       
    // 循环的获取BeanDefinition
    for(String beanDefinitionName : reader.getRegistry().getBeanDefinitionNames()){
        BeanDefinition definition = reader.getRegistry().getBeanDefinition(beanDefinitionName);
        System.out.println(definition);
        System.out.println(definition.getClass().getName());
    }
}
```
此外, 通过注解形式手动进行bean注册解析, 能够解析出 Bean 类上添加的一些关于BeanDefinition的注解,  `@ComponentScan` 就是基于这个做的扫描与注册, 底层的调用就是  `private <T> void AnnotatedBeanDefinitionReader#doRegisterBean` 方法, 创建一个 `AnnotationGenericBeanDefinition`  对象,  然后注册到内部的  `**BeanFactory**`** **工厂中** **中
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1660009546120-97cea824-8dca-41b6-9c00-0cc27b9d4f68.png#averageHue=%23323231&clientId=uc9d4667e-089e-4&from=paste&height=884&id=qemg2&originHeight=884&originWidth=1077&originalType=binary&ratio=1&rotation=0&showTitle=false&size=146666&status=done&style=none&taskId=ucf84307e-9538-4189-bd78-c0fdbf49e50&title=&width=1077)

看一下输出情况,  符合预期
```java
// bean 
@Component
@Primary
@Scope("prototype")
@Lazy
public class ParsedAnnotationBean{}

输出 >>> 
// BeanDefinition内容, 可以看到  scope、lazyInit、primary 等信息是配置的那样
Generic bean: class [org.cnc.spring.explain.configbean.MetaConfigBean]; scope=prototype; abstract=false; lazyInit=true; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=true; factoryBeanName=null; factoryMethodName=null; initMethodName=null; destroyMethodName=null
// BeanDefinition
org.springframework.beans.factory.annotation.AnnotatedGenericBeanDefinition
```

**需要特别关注 **
> 基于 手动注解注入无法完成依赖注入, 也就是说通过 @Resoure或者@Autowired 标记的属性无法正常注入,   具体原因是因为依赖注入是需要依靠 `AutowiredAnnotationBeanPostProcessor`  和 `CommonAnnotationBeanPostProcessor`两个 **BeanPostProcessor** 完成处理的

```java
public class ParsedAnnotationDependenceBean{}

public class ParsedAnnotationBean {
	@Autowired
	ParsedAnnotationDependenceBean parsedAnnotationDependenceBeanAutowired;
	@Resource
	ParsedAnnotationDependenceBean parsedAnnotationDependenceBeanResource;
}

public void testParseFromAnnotationConfiguration() {
    DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
    AnnotatedBeanDefinitionReader reader = new AnnotatedBeanDefinitionReader(factory);

    //注册注解处理器
    factory.addBeanPostProcessor(factory.getBean(CommonAnnotationBeanPostProcessor.class));
    factory.addBeanPostProcessor(factory.getBean(AutowiredAnnotationBeanPostProcessor.class));

    // 注册到 bean-factory
    reader.register(ParsedAnnotationDependenceBean.class);		//  被依赖bean
    reader.register(ParsedAnnotationBean.class); 				//  主bean
    
    ParsedAnnotationBean bean = factory.getBean(ParsedAnnotationBean.class);
}
```

#### 3. 基于API
基于API的方式则是通过手动创建一个 `BeanDefinition` 对象,注册到一个`BeanDefinitionRegistry` 中
```java
@Test
public void testConfigBeanFromBuilderApi() {
    // assign the bean name
    BeanDefinitionBuilder builder = BeanDefinitionBuilder.rootBeanDefinition(MetaConfigBean.class.getName());
    builder.addPropertyValue("name", "config from the builder API");
    BeanDefinition bd = builder.getBeanDefinition();
    // 创建spring 容器
    DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
    // 调用registryBeanDefinition向容器中注册bean
    final String beanName = "metaConfigApiBean";
    factory.registerBeanDefinition(beanName, bd);

    // 获取bean
    MetaConfigBean bean = factory.getBean(beanName, MetaConfigBean.class);
    System.out.println(bean);
}
```

## 阶段二、**Spring Bean注册阶段**
> **首先明确一点, SpringBean注册其实可以认为就是 **`**BeanDefinition**`**的注册; **
>

这里就会涉及到一个关键的接口: `BeanDefinitionRegistry`**,** Spring Bean注册将解析好的Bean注册到Bean注册器/Bean工厂内, 在Spring整体框架内一些核心的BeanFactory实现了BeanDefinitionRegistry接口, 代表支持将BeanDefinition注册到对应的BeanFactory容器中, 看一下类图
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661398128624-d025a8f7-9e99-46ab-b19f-9a81333f4ab9.png#averageHue=%232c2c2c&clientId=u9b97e286-4b34-4&from=paste&height=493&id=u14154405&originHeight=738&originWidth=1398&originalType=binary&ratio=1&rotation=0&showTitle=false&size=73831&status=done&style=none&taskId=u81eca684-d980-49dd-bfd6-db514b49c42&title=&width=934)
BeanDefinition 接口有三个直接的实现类

- **GenericApplicationContext**:  这个是主要核心 `SpringApplicationContext`  父类, 可以看到很多重要的ApplicationContext都实现了这个类,  但是这个类不对 BeanDefinitionRegistry做具体实现，BeanDefinition注册相关的能力都委托给 DefaultListableBeanFactory

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661398487529-8b3cc94c-0b4b-4595-a443-2807a2d14eaa.png#averageHue=%232d2c2c&clientId=u9b97e286-4b34-4&from=paste&height=135&id=ua79ed990&originHeight=135&originWidth=937&originalType=binary&ratio=1&rotation=0&showTitle=false&size=20455&status=done&style=none&taskId=u1fdc0de6-2a9e-4012-ac54-66860a063c3&title=&width=937)

- **DefaultListableBeanFactory** :  BeanDefinitionRegistry 的`唯一使用到`实现, BeanDefinitionRegistry的主要能力都由这个类实现
- **SimpleBeanDefinitionRegistry**:  只在测试使用
  :::success
  **结论:  DefaultListableBeanFactory 是 BeanDefinitionRegistry 的唯一实现**
  :::


**DefaultListableBeanFactory **实现了 BeanFactory 也印证了注册BeanDefinition相当于注册Bean的说法,  当然具体Bean的生成和存储还是会有更多细节； Spring官方对此也有佐证，比如基于 Annotation 配置的Bean最终会走到 `org.springframework.context.annotation.AnnotatedBeanDefinitionReader#doRegisterBean` 方法,   实际上方法内注册是 `**BeanDefinition**`
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661399175863-f2a74ffb-1a89-43f3-9b83-dc4b54f79608.png#averageHue=%232e2e2e&clientId=u9b97e286-4b34-4&from=paste&height=535&id=u6ab6390f&originHeight=535&originWidth=1069&originalType=binary&ratio=1&rotation=0&showTitle=false&size=53473&status=done&style=none&taskId=u4924dee3-14ae-4cb9-a666-74e7473f4a5&title=&width=1069)

测试下注册一个BeanDefinition 到 Registry, Factory有了BeanDefinition 就可以通过 getBean 去惰性的加载并获取一个Bean对象了
```java
@Test
public void testBeanDefinitionRegistry() {
    DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
    // 定义一个 bean definition
    GenericBeanDefinition gbd = new GenericBeanDefinition();
    gbd.setBeanClass(BeanDefinitionRegistryBean.class);
    // 注册
    beanFactory.registerBeanDefinition("bean", gbd);
    System.out.println(beanFactory.getBeanDefinition("bean"));
    System.out.println(beanFactory.containsBeanDefinition("bean"));
    System.out.println(Arrays.asList(beanFactory.getBeanDefinitionNames()));
    
    /**
    *  核心重点在 getBean的内部调用方法: {@link org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean}
    */
    BeanDefinitionRegistryBean bean = beanFactory.getBean(BeanDefinitionRegistryBean.class);
}
```

这个阶段在Spring的代码流程中，分别对应

- XML 配置： `org.springframework.context.support.AbstractApplicationContext#obtainFreshBeanFactory`
- Annotation配置 :  `org.springframework.context.annotation.AnnotatedBeanDefinitionReader#doRegisterBean`

有兴趣可以代码追踪, 最终会在  `org.springframework.context.support.AbstractApplicationContext#refresh` 这个方法集合
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661399904148-d934c4b1-a508-4da4-8624-6d8d8b686a4c.png#averageHue=%232e2c2c&clientId=u9b97e286-4b34-4&from=paste&height=567&id=u4cb101e1&originHeight=567&originWidth=1010&originalType=binary&ratio=1&rotation=0&showTitle=false&size=102313&status=done&style=none&taskId=u621d18bc-76ff-4e75-8dfe-3b88e185a53&title=&width=1010)

## 阶段三、**BeanDefinition合并阶段**
> 一些场景下, 可能存在一些父子Bean的场景, Bean的一些set配置存在父级时, 就需要对Bean进行一个父子合并，才能得到一个完整的子Bean对象, 这个阶段是将父bean的BeanDefinition与子 bean的BeanDefinition进行合并，最终得到一个包含完整信息的 RootBeanDefinition;

具体进行 BeanDefinition 合并的地方在 `org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean`
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661420518425-db676f3f-1b60-489f-952f-42a7bf9ed56e.png#averageHue=%232f2b2b&clientId=u9b97e286-4b34-4&from=paste&height=295&id=u95437b3b&originHeight=295&originWidth=1014&originalType=binary&ratio=1&rotation=0&showTitle=false&size=50302&status=done&style=none&taskId=u5f7c9297-5c87-4b35-8eb0-2e2c487c334&title=&width=1014)
下一级完成这个步骤的方法是: `org.springframework.beans.factory.support.AbstractBeanFactory#getMergedBeanDefinition`
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661420812969-317f9fa0-aea0-45f0-bd56-8e8387677bbb.png#averageHue=%232d2d2b&clientId=u9b97e286-4b34-4&from=paste&height=249&id=uf408ec97&originHeight=249&originWidth=846&originalType=binary&ratio=1&rotation=0&showTitle=false&size=47972&status=done&style=none&taskId=ud8014a0e-ee8a-4780-9a2c-f0ba851c6df&title=&width=846)
追踪往下可以看到完成的地方是在 `org.springframework.beans.factory.support.AbstractBeanDefinition#overrideFrom`, 很明显可以看出，所谓的合并BeanDefinition其实就是 子BeanDefinition 覆盖 父BeanDefinition的属性, 如: `Scope`,`Primary`等; 最终返回的父BeanDefinition就是完整的RootBeanDefinition;
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661420833521-46c4e4c0-250e-45fb-8d60-605377793469.png#averageHue=%232e2c2c&clientId=u9b97e286-4b34-4&from=paste&height=332&id=u967c9ca6&originHeight=332&originWidth=1088&originalType=binary&ratio=1&rotation=0&showTitle=false&size=59451&status=done&style=none&taskId=uc0a99245-5bd7-47c4-9352-9de8a935ea7&title=&width=1088)

> ⚠️  **注意 **:  很多时候, `AbstractBeanFactory#doGetBean`方法会作为创建bean的一个入口 , SpringBean  懒加载和非懒加载, 默认是非懒加载的
> - 非懒加载的 Bean 会在容器刷新阶段进行加载 , 具体地点是`org.springframework.context.support.AbstractApplicationContext#refresh`方法内对 `finishBeanFactoryInitialization(beanFactory)` 的调用, 深入查看可以看到这个方法内部最终还是会到  `doGetBean`方法


## 阶段四、**Bean Class加载阶段**
在`AbstractBeanDefinition` 中有一个属性 `private volatile Object beanClass`, 初始化时是一个字符串类型的BeanName, 在Bean的生命周期过程中, 通过 `AbstractBeanFactory#resolveBeanClass`方法将对应的BeanName转换为加载的Class对象, 然后设置到对应的BeanDefinition中去, 这个步骤发生在`AbstractAutowireCapableBeanFactory#createBean` 方法中
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1661422732010-46976dbc-18fd-4c4f-88d9-494bad0c66d2.png#averageHue=%232f2c2c&clientId=u9b97e286-4b34-4&from=paste&height=489&id=ueb66a486&originHeight=489&originWidth=958&originalType=binary&ratio=1&rotation=0&showTitle=false&size=78409&status=done&style=none&taskId=u7d46c7e7-3996-4e41-984d-5935c136eca&title=&width=958)

## 阶段五、Bean实例化过程
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


## 阶段六、**Bean属性设置阶段**
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

## 阶段七、**Bean初始化阶段**
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

## 阶段八、**所有单例bean初始化完成后阶段**
当所有的懒加载Bean都被加载完成后,Spring会遍历已经加载的Bean, 找出其中的  `SmartInitializingSingleton`,执行他们的 `afterSingletonsInstantiated`方法;   这个步骤的入口在 `DefaultListableBeanFactory#preInstantiateSingletons`
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1662044621649-d4c5fea4-78bb-45c3-9e62-a417ec085eb0.png#averageHue=%23262626&clientId=ue77b7ba8-4cd9-4&from=paste&height=401&id=ua21bab27&originHeight=401&originWidth=990&originalType=binary&ratio=1&rotation=0&showTitle=false&size=77476&status=done&style=none&taskId=u7552ec18-0d12-4e12-a0f0-13fe96c3be4&title=&width=990)
> 可以在这个阶段进行一些通知通告的调用,或者一些缓存的加载了,  因为到了这里基本上是确定所有的Bean都已经初始化完成

## 阶段九、**Bean销毁阶段**
从源码角度来说, 触发SpringBean进行销毁的场景有以下两种

- 调用 `AbstractAutowireCapableBeanFactory#destroyBean`

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1662286513459-3b7de20c-90ea-4b98-a011-7a088a919160.png#averageHue=%23262525&clientId=ub2e6a66f-7d25-4&from=paste&height=146&id=u26982e82&originHeight=146&originWidth=949&originalType=binary&ratio=1&rotation=0&showTitle=false&size=19737&status=done&style=none&taskId=u36679dc1-72ac-4caa-8d81-f5596716d91&title=&width=949)

- 调用`ConfigurableBeanFactory#destroySingletons` 前文说过,  这个只有一个最终实现就是, **DefaultListableBeanFactory,  **同时还有一个`AbstractApplicationContext#close`,  但是这个方法最终还会回到`ConfigurableBeanFactory#destroySingletons`

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1662286519607-c9a7e129-9bd8-4962-bcfe-5d0f4f0caf4a.png#averageHue=%23282726&clientId=ub2e6a66f-7d25-4&from=paste&height=174&id=u28b2c036&originHeight=174&originWidth=712&originalType=binary&ratio=1&rotation=0&showTitle=false&size=21106&status=done&style=none&taskId=ud4b59101-39ae-409f-95d1-0b3585726de&title=&width=712)
### 1. AbstractAutowireCapableBeanFactory#destroyBean
基于这个流程的最终调用会到  `DisposableBeanAdapter#destroy`
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1662300064171-c6b4c00a-41f3-43c9-a4ea-744d1b186e52.png#averageHue=%23333232&clientId=ub2e6a66f-7d25-4&from=paste&height=147&id=ud967bcfb&originHeight=147&originWidth=903&originalType=binary&ratio=1&rotation=0&showTitle=false&size=18610&status=done&style=none&taskId=u5fde3550-2c74-41c4-842a-c5b96b4e34b&title=&width=903)
### 2. ConfigurableBeanFactory#destroySingletons
基于`ConfigurableBeanFactory`的`bean`销毁调用路径如下(从context.close() 开始):

1. `AbstractApplicationContext#doClose`
2. `AbstractApplicationContext#destroyBeans`
3. `DefaultListableBeanFactory#destroySingletons`
4. `DefaultSingletonBeanRegistry#destroySingletons`
5. `DefaultSingletonBeanRegistry#destroySingleton`
6. `DefaultSingletonBeanRegistry#destroyBean`
7. `DisposableBean#destroy`
> 跟进发现其实基于 ConfigurableBeanFactory的调用最终会调用到  `DisposableBeanAdapter#destroy`, 但是两种调用路径的区别在于: `AbstractAutowireCapableBeanFactory#destroyBean`直接调用的生成的 `DisposableBeanAdapter#destroy` 没有`destoryMethod`,导致不能调用自定义的销毁方法

### DisposableBeanAdapter#destroy 调用的逻辑说明
#### 第一步、调用DestructionAwareBeanPostProcessor的postProcessBeforeDestruction
这个阶段主要是执行了关键一个Processor: **InitDestroyAnnotationBeanPostProcessor****,  **这个Processor在销毁阶段会执行 `@PreDestroy` 这个注解的方法
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1662302326571-82401d5a-dedd-47cf-9c8f-3d1d4b4f0ece.png#averageHue=%232c2c2b&clientId=ub2e6a66f-7d25-4&from=paste&height=305&id=ud2dcf72b&originHeight=305&originWidth=1160&originalType=binary&ratio=1&rotation=0&showTitle=false&size=60094&status=done&style=none&taskId=udd93e8d4-4a71-4d1b-af44-13fde117b09&title=&width=1160)
同时,也可以自定义实现` DestructionAwareBeanPostProcessor#postProcessBeforeDestruction` 进行相关销毁拦截

小小测试一下
```java
@Test
public void testDestructionAwareBeanPostProcessor() {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
    ctx.refresh();
    ctx.getBeanFactory().addBeanPostProcessor(new DestructionAwareBeanPostProcessor() {
        @Override
        public void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException {
            if (DestructionAwareBeanPostProcessorBean.class.equals(bean.getClass())) {
                System.out.println("执行 DestructionAwareBeanPostProcessorBean 的销毁");
            }
        }
    });
    ctx.register(DestructionAwareBeanPostProcessorBean.class);
    ctx.getBean(DestructionAwareBeanPostProcessorBean.class);
    ctx.close();
}
// 输出
执行 DestructionAwareBeanPostProcessorBean 的销毁
```

#### 第二步、 执行DisposableBean的destory
这个阶段的执行源码如下:
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22746802/1662302529435-c9176f48-7456-4943-8021-7dab8a381656.png#averageHue=%232c2b2b&clientId=ub2e6a66f-7d25-4&from=paste&height=567&id=u63f0a4f2&originHeight=567&originWidth=969&originalType=binary&ratio=1&rotation=0&showTitle=false&size=78793&status=done&style=none&taskId=u492f428d-ab42-441c-a8b9-e91e0a39e75&title=&width=969)
通过一个简单的例子看下
```java
public class BeanDestroyBean implements DisposableBean {
	@Override
	public void destroy() throws Exception {
		System.out.println("DisposableBean 执行");
	}

	@PreDestroy
	public void preDestroy(){
		System.out.println("PreDestroy执行");
	}
}

@Test
public void testDestroyBean() {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
    ctx.refresh();
    ctx.register(BeanDestroyBean.class);
    ctx.getBean(BeanDestroyBean.class);
    ctx.close();
}
// 输出
PreDestroy执行
DisposableBean 执行
```
#### 第三步、执行自定义的销毁方法
这个步骤就是执行Bean的自定义的一些销毁方法, 定义销毁方法的方式和定义初始化方法一致, 这里针对基于  `@Bean`  的方式来进行简单示例
```java
public class BeanCustomBeanDestroy {
	public void customDestroy(){
		System.out.println("执行 自定义销毁方法");
	}
}
@Configuration
public class BeanDestroyConfiguration {
	@Bean(destroyMethod = "customDestroy")
	public BeanCustomBeanDestroy beanCustomBeanDestroy(){
		return new BeanCustomBeanDestroy();
	}
}
	@Test
public void testCustomDestroyMethod(){
		AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(BeanDestroyConfiguration.class);;
		ctx.close();
}
//输出
执行 自定义销毁方法
```


# 第三部分、生命周期钩子
> SpringBean 生命周期第[1]步: 		执行 InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation 回调
> SpringBean 生命周期第[2]步: 		执行 SmartInstantiationAwareBeanPostProcessor#determineCandidateConstructors回调
> SpringBean 生命周期第[3]步: 		执行 MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition 回调
> SpringBean 生命周期第[4]步: 		执行 InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation 回调
> SpringBean 生命周期第[5]步: 		执行 InstantiationAwareBeanPostProcessor#postProcessProperties 回调
> SpringBean 生命周期第[6]步: 		执行 CommonAnnotationBeanPostProcessor#postProcessProperties 回调, 处理JSR-250自动注入
> SpringBean 生命周期第[7]步: 		执行 AutowiredAnnotationBeanPostProcessor#postProcessProperties 回调, 处理Spring自动注入
> SpringBean 生命周期第[8]步: 		执行  BeanNameAware#setBeanName 回调
> SpringBean 生命周期第[9]步: 		执行  BeanClassLoaderAware#setBeanClassLoader 回调
> SpringBean 生命周期第[10]步: 	执行  BeanFactoryAware#beanFactory 回调
> SpringBean 生命周期第[11]步: 	执行  ApplicationContextAware#setApplicationContext 回调
> SpringBean 生命周期第[12]步: 	执行 BeanPostProcessor#postProcessBeforeInitialization 回调
> SpringBean 生命周期第[13]步: 	执行 InitDestroyAnnotationBeanPostProcessor#postProcessBeforeInitialization  @PostConstruct  回调
> SpringBean 生命周期第[14]步: 	执行  InitializingBean#afterPropertiesSet 回调
> SpringBean 生命周期第[15]步: 	执行 自定义初始化方法 回调
> SpringBean 生命周期第[16]步: 	执行 BeanPostProcessor#postProcessAfterInitialization 回调
> SpringBean 生命周期第[17]步: 	执行 DestructionAwareBeanPostProcessor#postProcessBeforeDestruction 回调
> SpringBean 生命周期第[18]步: 	执行  InitDestroyAnnotationBeanPostProcessor.postProcessBeforeDestruction @PreDestroy 回调
> SpringBean 生命周期第[19]步: 	执行  DisposableBean#destroy 回调
> SpringBean 生命周期第[20]步: 	执行 自定义销毁方法 回调


![生命周期钩子.jpg](https://cdn.nlark.com/yuque/0/2022/jpeg/22746802/1663207632406-fe82d053-6443-41b6-bf3b-e1cc6fa231f9.jpeg#averageHue=%2399cdac&clientId=u9a608823-9706-4&from=ui&id=u772b3322&originHeight=3788&originWidth=4401&originalType=binary&ratio=1&rotation=0&showTitle=false&size=824507&status=done&style=none&taskId=ucf0061e8-07a8-4c52-b663-87889a22900&title=)


