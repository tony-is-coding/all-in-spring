# Table of Contents

  * [**Bean销毁阶段**](#bean销毁阶段)
    * [1. AbstractAutowireCapableBeanFactory#destroyBean](#1-abstractautowirecapablebeanfactorydestroybean)
    * [2. ConfigurableBeanFactory#destroySingletons](#2-configurablebeanfactorydestroysingletons)
    * [DisposableBeanAdapter#destroy 调用的逻辑说明](#disposablebeanadapterdestroy-调用的逻辑说明)
      * [第一步、调用DestructionAwareBeanPostProcessor的postProcessBeforeDestruction](#第一步调用destructionawarebeanpostprocessor的postprocessbeforedestruction)


## **Bean销毁阶段**
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
这个阶段主要是执行了关键一个Processor: **InitDestroyAnnotationBeanPostProcessor**
