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
![img.png](src/i2.png)
