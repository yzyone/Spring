# Spring Boot 项目开机自启动设置

**1.前提**

　　linux已安装JDK和MySQL。并记录JDK安装路径。如何查看JDK安装路径，请参考https://www.cnblogs.com/shizhe99/p/14595881.html。

**2.将项目打包成jar包文件**

　　上传jar包到某路径下，例如：/usr/local/project/。请记住本路径稍后会用到。可是试运行jar包，看是否能正常启动。

```
　　java -jar springboot.jar
```

**3.新增service文件**

　　在 /etc/systemd/system/ 目录下面编辑一个以service为后缀的文件：

```
　　 cd /etc/systemd/system 

　　 vi java.service 
```

　　文件内容如下：



```
1 [Unit]
2 Description=java
3 After=syslog.target
4 [Service]
5 Type=simple
6 ExecStart=/usr/jdk1.8.0_281/bin/java -jar /usr/local/project/springboot.jar
7 [Install]
8 WantedBy=multi-user.target
```

　　Description为描述，大家可以对自己的项目进行相应的描述即可。

　　ExecStart前面为JDK安装路径，后面为jar包路径。

　　java.service大家可以将java替换为相应的名称。

**4.添加执行权限**

```
　　 chmod +x /etc/systemd/system/java.service
```

**5.重新加载服务**

```
　　 systemctl daemon-reload 
```

**6.启动服务并加入开机自启动**



```
　　 systemctl start java
　　 systemctl enable java.service
```

　　本步骤中的java、java.service大家替换为自己取得名称即可。

 

**注：**



```
　　 ps -ef | grep "java"| grep -v grep
　　 systemctl status java.service -l
```

上面的命令查看当前java运行状态。

下面的用来查看服务的状态。