启动流程

1. 首先从main找到run()方法，在执行run()方法之前new一个SpringApplication对象

2. 进入run()方法，创建应用监听器SpringApplicationRunListeners开始监听
3. 然后加载SpringBoot配置环境(ConfigurableEnvironment)，然后把配置环境(Environment)加入监听对象中
4. 然后加载应用上下文(ConfigurableApplicationContext)，当做run方法的返回对象
5. 最后创建Spring容器，refreshContext(context)，实现starter自动化配置和bean的实例化等工作。