# Springboot命令行配置--spring.config.location配置文件的优先级

讨论一个springboot的配置
--spring.config.location=C:/application.properties

这个配置只能用在命令行里，写在配置文件里是无效的，主要用来运维已经打包好了的程序，想要指定配置文件路径的情况

示例：

`java -jar aaa.jar --spring.config.location=C:/application.properties`

那么会有一个问题

当这里面的配置与springboot，jar包里的配置文件配置冲突会怎么办？
先贴一下，springboot的配置文件加载位置与优先级

springboot 启动会扫描以下位置的application.properties或者application.yml文件作为Spring boot的默认配置文件

`–file:./config/`

`–file:./`

`–classpath:/config/`

`–classpath:/`

优先级由高到底，高优先级的配置会覆盖低优先级的配置；

SpringBoot会从这四个位置全部加载主配置文件；互补配置；

如果配置文件里配置了激活文件，或者代码里有加载别的配置文件等，全部的加载顺序优先级从上往下，依次递减

`spring.profiles.active=test`

那么 --spring.config.location=C:/application.properties 配置的文件，它的优先级在哪里呢，经过试验与查阅官方文档，总结了springboot的配置加载顺序。

 

==SpringBoot也可以从以下位置加载配置； 优先级从高到低；高优先级的配置覆盖低优先级的配置，所有的配置会形成互补配置==

1.命令行参数

所有的配置都可以在命令行上进行指定

`java -jar spring-boot-02-config-02-0.0.1-SNAPSHOT.jar --server.port=8087 --server.context-path=/abc`

多个配置用空格分开； --配置项=值

2.来自java:comp/env的JNDI属性

3.Java系统属性（System.getProperties()）

4.操作系统环境变量

5.RandomValuePropertySource配置的random.*属性值

==由jar包外向jar包内进行寻找；==（*.properties>*.yml）

==优先加载带profile==

6.jar包外部的application-{profile}.properties或application.yml(带spring.profile)配置文件

7.jar包内部的application-{profile}.properties或application.yml(带spring.profile)配置文件

8.--spring.config.location=C:/application.properties（它在这里）

==再来加载不带profile==

9.jar包外部的application.properties或application.yml(不带spring.profile)配置文件

10.jar包内部的application.properties或application.yml(不带spring.profile)配置文件

11.@Configuration注解类上的@PropertySource

12.通过SpringApplication.setDefaultProperties指定的默认属性

————————————————

版权声明：本文为CSDN博主「六月·飞雪」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://blog.csdn.net/z_ssyy/article/details/105347680