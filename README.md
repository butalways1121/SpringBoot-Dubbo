# SpringBoot-Dubbo
SpringBoot之Dubbo
---
&emsp;&emsp;**本文介绍了Windows下Zookeeper单机环境的安装、dubbo-admin基于jdk1.8 & Tomcat 7的部署，以及利用Zookeeper做注册中心整合SpringBoot和Dubbo来发布服务，项目源码戳[这里](https://github.com/butalways1121/SpringBoot-Dubbo)。**
<!--more-->
## 一、Dubbo与Zookeeper
### Dubbo
#### Dubbo是什么
&emsp;&emsp;当服务越来越多时，容量的评估、小服务资源的浪费等问题逐渐显现，此时需要增加一个调度中心基于访问压力实时管理集群容量，提供集群利用率。其中，用于提高机器利用率的资源调度和治理中心是关键。

&emsp;&emsp;Dubbo是阿里巴巴开源项目的一个分布式服务框架，致力于提供高性能和透明化的RPC远程服务调用方案，以及SOA服务治理方案。简单的说，Dubbo就是个服务框架，如果没有分布式的需求，其实是不需要用的，只有在分布式的时候，才有Dubbo这样的分布式服务框架的需求，并且本质上是个服务调用的东西，说白了就是个远程服务调用的分布式框架（告别Web Service模式中的WSDL，以服务者与消费者的方式在Dubbo上注册）。
***
**名词解释：**
&emsp;&emsp;WSDL：Web Services Description Language，网络服务描述语言
&emsp;&emsp;RPC：Remote Procedure Call Protocol，远程过程调用协议
&emsp;&emsp;SOA：Service-Oriented Architecture，面向服务的体系结构
#### Dubbo的工作原理
如图：

![](https://gitee.com/butalways1121/img-Blog/raw/master/106.png)
**角色节点说明：**
&emsp;&emsp;Provider: 暴露服务的服务提供方；
&emsp;&emsp;Consumer: 调用远程服务的服务消费方；
&emsp;&emsp;Registry: 服务注册与发现的注册中心；
&emsp;&emsp;Monitor: 统计服务的调用次调和调用时间的监控中心；
&emsp;&emsp;Container: 服务运行容器。
**调用关系说明：**
&emsp;&emsp;0.服务器启动、加载和运行；
&emsp;&emsp;1.服务提供者在启动时，向注册中心注册自己提供的服务；
&emsp;&emsp;2.服务消费者在启动时，向注册中心订阅自己所需的服务；
&emsp;&emsp;3.注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更给消费者；
&emsp;&emsp;4.服务消费者从地址列表中，基于软负载均衡算法选一台服务提供者进行调用，如果调用失败再选另一台；
&emsp;&emsp;5.服务消费者和服务提供者在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。
### Zookeeper
#### Zookeeper是什么
&emsp;&emsp;ZooKeeper是一个分布式的，开放源码的分布式应用程序协调服务，是Google的Chubby一个开源的实现，是Hadoop和Hbase的重要组件。它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等。
#### Zookeeper的作用
&emsp;&emsp;Zookeeper作为一个分布式的服务框架，主要用来解决分布式集群中应用系统的一致性问题，它能提供基于类似于文件系统的目录节点树方式的数据存储，但是Zookeeper并不是用来专门存储数据的，它的作用主要是用来维护和监控你存储的数据的状态变化，通过监控这些数据状态的变化，从而可以达到基于数据的集群管理。
### Zookeeper和Dubbo的关系
&emsp;&emsp;Dubbo将注册中心进行抽象，使得它可以外接不同的存储媒介给注册中心提供服务，有ZooKeeper、Memcached、Redis等。简单来说打个比方：Dubbo就是动物园的动物，Zookeeper是动物园。如果游客想看动物的话就要去动物园。换句话说就是要把很多不同的Dubbo（动物）放到Zookeeper（动物园中）提供给游客进行观赏。

&emsp;&emsp;引入ZooKeeper作为存储媒介，也就把ZooKeeper的特性引进来。首先是负载均衡，单注册中心的承载能力是有限的，在流量达到一定程度的时候就需要分流，负载均衡就是为了分流而存在的，一个ZooKeeper群配合相应的Web应用就可以很容易达到负载均衡；资源同步，单单有负载均衡还不够，节点之间的数据和资源需要同步，ZooKeeper集群就天然具备有这样的功能；命名服务，将树状结构用于维护全局的服务地址列表，服务提供者在启动的时候，向Zookeeper上的指定节点/dubbo/${serviceName}/providers目录下写入自己的URL地址，这个操作就完成了服务的发布。其他特性还有Mast选举，分布式锁等。
## 二、Zookeeper环境搭建
&emsp;&emsp;因为后面要使用Zookeeper注册中心，所以首先需要搭建Zookeeper的环境。Zookeeper支持Windows、Linux、Mac等操作系统，其搭建方式也有集群、伪集群、单机环境，这里我们只简单的搭建Windows操作系统的单机环境即可，不过Windows下的环境适合做开发，但是不适合生产环境的部署安装。
### 1.Java环境
&emsp;&emsp;Zookeeper依赖于Java环境，所以需要先行安装jdk，这里就不再多说了。
### 2.Zookeeper的下载及安装
&emsp;&emsp;首先，下载zookeeperxxx.tar.gz，地址：`https://archive.apache.org/dist/zookeeper/zookeeper-3.4.14/zookeeper-3.4.14.tar.gz
`，完成之后解压即可。

&emsp;&emsp;在解压之后的`conf`目录下创建`zoo.cfg`文件（该目录下提供了一分样本文件`zoo_sample.cfg`，将其复制为`zoo.cfg`也可），主要内容如下：
```bash
tickTime=2000
initLimit=10
syncLimit=5
dataDir=D:/zookeeper-3.4.14/data
clientPort=2181
```
**解释：**
&emsp;&emsp;`tickTime`是作为Zookeeper服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个tickTime时间就会发送一个心跳；
&emsp;&emsp;`initLimit`是集群中的follower服务器(F)与leader服务器(L)之间初始连接时能容忍的最多心跳数（tickTime的数量）；
&emsp;&emsp;`syncLimit`表示集群中的follower服务器(F)与leader服务器(L)之间请求和应答之间能容忍的最多心跳数（tickTime的数量）；
&emsp;&emsp;`dataDir`是Zookeeper保存数据的目录，默认情况下，Zookeeper将写数据的日志文件也保存在这个目录里，指定了该路径之后需要对应着进行目录的创建;
&emsp;&emsp;`clientPort`是客户端连接Zookeeper服务器的端口，Zookeeper会监听这个端口，接受客户端的访问请求。

&emsp;&emsp;接着，双击Zookeeper安装目录的bin文件下的`zkServer.cmd`来启动Zookeeper，输出信息如下：
```bash
D:\zookeeper-3.4.14\bin>call "D:\Program Files\Java\jdk1.8.0_211"\bin\java "-Dzo
okeeper.log.dir=D:\zookeeper-3.4.14\bin\.." "-Dzookeeper.root.logger=INFO,CONSOL
E" -cp "D:\zookeeper-3.4.14\bin\..\build\classes;D:\zookeeper-3.4.14\bin\..\buil
d\lib\*;D:\zookeeper-3.4.14\bin\..\*;D:\zookeeper-3.4.14\bin\..\lib\*;D:\zookeep
er-3.4.14\bin\..\conf" org.apache.zookeeper.server.quorum.QuorumPeerMain "D:\zoo
keeper-3.4.14\bin\..\conf\zoo.cfg"
2019-12-03 14:29:47,504 [myid:] - INFO  [main:QuorumPeerConfig@136] - Reading co
nfiguration from: D:\zookeeper-3.4.14\bin\..\conf\zoo.cfg
2019-12-03 14:29:47,551 [myid:] - INFO  [main:DatadirCleanupManager@78] - autopu
rge.snapRetainCount set to 3
2019-12-03 14:29:47,552 [myid:] - INFO  [main:DatadirCleanupManager@79] - autopu
rge.purgeInterval set to 0
2019-12-03 14:29:47,552 [myid:] - INFO  [main:DatadirCleanupManager@101] - Purge
 task is not scheduled.
2019-12-03 14:29:47,555 [myid:] - WARN  [main:QuorumPeerMain@116] - Either no co
nfig or no quorum defined in config, running  in standalone mode
2019-12-03 14:29:47,901 [myid:] - INFO  [main:QuorumPeerConfig@136] - Reading co
nfiguration from: D:\zookeeper-3.4.14\bin\..\conf\zoo.cfg
2019-12-03 14:29:47,902 [myid:] - INFO  [main:ZooKeeperServerMain@98] - Starting
 server
2019-12-03 14:29:48,005 [myid:] - INFO  [main:Environment@100] - Server environm
ent:zookeeper.version=3.4.14-4c25d480e66aadd371de8bd2fd8da255ac140bcf, built on
03/06/2019 16:18 GMT
2019-12-03 14:29:48,005 [myid:] - INFO  [main:Environment@100] - Server environm
ent:host.name=A06824.jac.net
2019-12-03 14:29:48,006 [myid:] - INFO  [main:Environment@100] - Server environm
ent:java.version=1.8.0_211
2019-12-03 14:29:48,007 [myid:] - INFO  [main:Environment@100] - Server environm
ent:java.vendor=Oracle Corporation
2019-12-03 14:29:48,007 [myid:] - INFO  [main:Environment@100] - Server environm
ent:java.home=D:\Program Files\Java\jdk1.8.0_211\jre
2019-12-03 14:29:48,008 [myid:] - INFO  [main:Environment@100] - Server environm
ent:java.class.path=D:\zookeeper-3.4.14\bin\..\build\classes;D:\zookeeper-3.4.14
\bin\..\build\lib\*;D:\zookeeper-3.4.14\bin\..\zookeeper-3.4.14.jar;D:\zookeeper
-3.4.14\bin\..\lib\audience-annotations-0.5.0.jar;D:\zookeeper-3.4.14\bin\..\lib
\jline-0.9.94.jar;D:\zookeeper-3.4.14\bin\..\lib\log4j-1.2.17.jar;D:\zookeeper-3
.4.14\bin\..\lib\netty-3.10.6.Final.jar;D:\zookeeper-3.4.14\bin\..\lib\slf4j-api
-1.7.25.jar;D:\zookeeper-3.4.14\bin\..\lib\slf4j-log4j12-1.7.25.jar;D:\zookeeper
-3.4.14\bin\..\conf
2019-12-03 14:29:48,010 [myid:] - INFO  [main:Environment@100] - Server environm
ent:java.library.path=D:\Program Files\Java\jdk1.8.0_211\bin;C:\Windows\Sun\Java
\bin;C:\Windows\system32;C:\Windows;D:\Program Files (x86)\Integrity\ILMClient12
\bin;E:\app\20180877\product\12.1.0\client_1\bin;D:\app;C:\Windows\system32;C:\W
indows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;D:\P
rogram Files\Java\jdk1.8.0_211\bin;D:\Program Files\Java\jdk1.8.0_211\jre\bin;D:
\Program Files\TortoiseSVN\bin;D:\Program Files\nodejs\;D:\Program Files\Git\cmd
;D:\apache-maven-3.6.1;D:\apache-maven-3.5.4\bin;C:\Users\20180877\AppData\Roami
ng\npm\node_modules\cnpm\bin;F:\blog\node_modules\.bin;D:\Python;D:\Python\Scrip
ts;D:\PROGRA~2\INTEGR~1\Toolkit\mksnt;D:\apache-tomcat-7.0.96\bin;D:\apache-tomc
at-7.0.96\lib;D:\apache-tomcat-7.0.96\bin;D:\Program Files\erl10.5\bin;D:\Progra
m Files\RabbitMQ Server\rabbitmq_server-3.8.1\sbin;C:\Program Files\MySQL\MySQL
Shell 8.0\bin\;D:\Bandizip\;D:\Users\20180877\AppData\Local\Programs\Microsoft V
S Code\bin;.
2019-12-03 14:29:48,013 [myid:] - INFO  [main:Environment@100] - Server environm
ent:java.io.tmpdir=C:\Users\20180877\AppData\Local\Temp\
2019-12-03 14:29:48,014 [myid:] - INFO  [main:Environment@100] - Server environm
ent:java.compiler=<NA>
2019-12-03 14:29:48,017 [myid:] - INFO  [main:Environment@100] - Server environm
ent:os.name=Windows 7
2019-12-03 14:29:48,018 [myid:] - INFO  [main:Environment@100] - Server environm
ent:os.arch=amd64
2019-12-03 14:29:48,021 [myid:] - INFO  [main:Environment@100] - Server environm
ent:os.version=6.1
2019-12-03 14:29:48,022 [myid:] - INFO  [main:Environment@100] - Server environm
ent:user.name=20180877
2019-12-03 14:29:48,024 [myid:] - INFO  [main:Environment@100] - Server environm
ent:user.home=C:\Users\20180877
2019-12-03 14:29:48,025 [myid:] - INFO  [main:Environment@100] - Server environm
ent:user.dir=D:\zookeeper-3.4.14\bin
2019-12-03 14:29:48,059 [myid:] - INFO  [main:ZooKeeperServer@836] - tickTime se
t to 2000
2019-12-03 14:29:48,060 [myid:] - INFO  [main:ZooKeeperServer@845] - minSessionT
imeout set to -1
2019-12-03 14:29:48,061 [myid:] - INFO  [main:ZooKeeperServer@854] - maxSessionT
imeout set to -1
2019-12-03 14:29:48,660 [myid:] - INFO  [main:ServerCnxnFactory@117] - Using org
.apache.zookeeper.server.NIOServerCnxnFactory as server connection factory
2019-12-03 14:29:48,667 [myid:] - INFO  [main:NIOServerCnxnFactory@89] - binding
 to port 0.0.0.0/0.0.0.0:2181
```
### 3.测试
&emsp;&emsp;启动之后在cmd下输入`netstat -ano|findstr 2181`或`jps -l`来查看Zookeeper是否启动成功：
```bash
C:\Users\20180877>netstat -ano|findstr 2181
  TCP    0.0.0.0:2181           0.0.0.0:0              LISTENING       9060
  TCP    [::]:2181              [::]:0                 LISTENING       9060

C:\Users\20180877>jps -l
7504 sun.tools.jps.Jps
9060 org.apache.zookeeper.server.quorum.QuorumPeerMain
```
## 三、Dubbo-admin的部署
&emsp;&emsp;首先，在git上下载dubbo工程，地址：`https://github.com/apache/incubator-dubbo`，里面包含很多项目,我们只需要用到dubbo-admin就可以了，**温馨提示，master分支上是没有dubbo-admin模块的，所以下载之前需将其切换至2.5.X分支**，如图：
![](https://gitee.com/butalways1121/img-Blog/raw/master/107.png)

&emsp;&emsp;下载完成之后，进入dubbo-admin目录，在该目录下打开cmd，运行`mvn package -Dmaven.skip.test=true`命令将其打成war包，命令执行完成后如图，其中有显示war包的存放位置：

![](https://gitee.com/butalways1121/img-Blog/raw/master/108.png)

&emsp;&emsp;接着，将该war包存放于Tomcat的webapps目录下，如`D:\apache-tomcat-7.0.96\webapps，`，再运行`D:\apache-tomcat-7.0.96\bin\startup.bat`启动Tomcat。

&emsp;&emsp;启动之后，在浏览器中输入`localhost:8080/dubbo-admin-2.5.10`即可进行管理，登录的用户名和密码默认都是root:

![](https://gitee.com/butalways1121/img-Blog/raw/master/109.png)
![](https://gitee.com/butalways1121/img-Blog/raw/master/110.png)

## 四、SpringBoot整合Dubbo示例
本次项目分为服务端和客户端两个：
### 1.Dubbo服务端
&emsp;&emsp;新建SpringBootDubboServer项目作为Dubbo服务的提供者，pom.xml文件配置如下：
```bash
<?xml version="1.0"?>
<project
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd"
	xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.2.RELEASE</version>
	</parent>
	<groupId>cn.qlq</groupId>
	<artifactId>springboot-dubbo-server</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>springboot-dubbo-server</name>
	<url>http://maven.apache.org</url>
	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
	</properties>
	<dependencies>
		<!-- dubbo的依赖 -->
		<dependency>
			<groupId>com.alibaba.spring.boot</groupId>
			<artifactId>dubbo-spring-boot-starter</artifactId>
			<version>2.0.0</version>
		</dependency>
		<!-- zookeeper client依赖 -->
		<dependency>
			<groupId>com.101tec</groupId>
			<artifactId>zkclient</artifactId>
			<version>0.9</version>
		</dependency>
	</dependencies>

	<build>
		<!-- 配置了很多插件 -->
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<configuration>
					<source>1.7</source>
					<target>1.7</target>
					<encoding>UTF-8</encoding>
				</configuration>
			</plugin>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>
```
application.properties文件如下：
```bash
# DUBBO相关配置
#当前服务/应用名称
spring.dubbo.application.name=provider
#注册中心的协议和地址
spring.dubbo.server=true
spring.dubbo.registry=zookeeper://127.0.0.1:2181
#通信规则(通信协议和接口)
spring.dubbo.protocol.name=dubbo
spring.dubbo.protocol.port=20880
```
&emsp;&emsp;为了便于进行客户端调用测试这里创建一个User类，属性包括userName和password，并创建相应的service服务及其实现类来对User进行赋值。

User类代码：
```bash
import java.io.Serializable;

public class User implements Serializable {
	private static final long serialVersionUID = 8671229885144085830L;
	private String userName;
	private String password;

	public String getUserName() {
		return userName;
	}

	public void setUserName(String userName) {
		this.userName = userName;
	}

	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		this.password = password;
	}

	public User(String userName, String password) {
		super();
		this.userName = userName;
		this.password = password;
	}
}
```
servive层：
```bash
import java.util.List;

import org.springboot.dubbo.user.User;

public interface UserService {
	List<User> getAllUsers();
}
```
service的实现类代码：
```bash
import java.util.ArrayList;
import java.util.List;

import org.springboot.dubbo.service.UserService;
import org.springboot.dubbo.user.User;
import org.springframework.stereotype.Component;

import com.alibaba.dubbo.config.annotation.Service;

@Service(version = "1.0.0") 
@Component
public class UserServiceImpl implements UserService {

	public List<User> getAllUsers() {
		List<User> users = new ArrayList<User>();
		for (int i = 0; i < 20; i++) {
			User user = new User("username" + i, "password" + i);
			users.add(user);
		}

		return users;
	}
}
```
**注：这里的`@Service`注解是Dubbo提供的，表示Dubbo提供者服务用于声明对外暴露服务,其中的version用于指定服务的版本号。**
启动类代码：
```bash
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

import com.alibaba.dubbo.spring.boot.annotation.EnableDubboConfiguration;

@SpringBootApplication
@EnableDubboConfiguration
public class DubboServerApplication {
	public static void main(String[] args) throws InterruptedException {
		SpringApplication.run(DubboServerApplication.class, args);
	}
}
```
其中，`@EnableDubboConfiguration`注解表示要开启Dubbo功能。
### 2.Dubbo客户端
&emsp;&emsp;新建SpringBootDubboClient项目来使用Dubbo提供的服务，pom.xml文件配置同服务端即可。
application.properties文件：
```bash
# DUBBO相关配置
#当前服务/应用名称
spring.dubbo.application.name=consumer
#注册中心的协议和地址
spring.dubbo.server=true
spring.dubbo.registry=zookeeper://127.0.0.1:2181
#通信规则(通信协议和接口)
spring.dubbo.protocol.name=dubbo
spring.dubbo.protocol.port=20880
spring.dubbo.scan=org.springboot.dubbo
```
同样，创建User实体类：
```bash
import java.io.Serializable;

public class User implements Serializable {
	private static final long serialVersionUID = 8671229885144085830L;
	private String userName;
	private String password;

	public String getUserName() {
		return userName;
	}

	public void setUserName(String userName) {
		this.userName = userName;
	}

	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		this.password = password;
	}

	public User(String userName, String password) {
		super();
		this.userName = userName;
		this.password = password;
	}
}
```
service层代码：
```bash
import java.util.List;

import org.springboot.dubbo.user.User;

public interface UserService {
	List<User> getAllUsers();
}
```
controller层代码示例：
```bash
import java.util.List;

import org.springboot.dubbo.service.UserService;
import org.springboot.dubbo.user.User;
import org.springframework.stereotype.Controller;

import com.alibaba.dubbo.config.annotation.Reference;

@Controller
public class UserController {
	@Reference(version = "1.0.0")
	UserService userService;

	public List<User> getAllUsers() {
		return userService.getAllUsers();
	}
}
```
其中，`@Reference`注解用于Dubbo消费者服务指明引用哪个提供者接口服务。
启动类代码：
```bash
import java.util.List;

import org.springboot.dubbo.controller.UserController;
import org.springboot.dubbo.user.User;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;

import com.alibaba.dubbo.spring.boot.annotation.EnableDubboConfiguration;

@SpringBootApplication
@EnableDubboConfiguration
public class DubboClientApplication {
	public static void main(String[] args) {
		// 入口运行类
		//启动嵌入式的 Tomcat 并初始化 Spring 环境及其各 Spring 组件
		ConfigurableApplicationContext run = SpringApplication.run(DubboClientApplication.class, args);
		System.out.println("DubboClient启动成功。。。");
		UserController bean = run.getBean(UserController.class);
		List<User> allUsers = bean.getAllUsers();
		for(User user:allUsers) {
			System.out.println("用户名: "+user.getUserName()+"，密码: "+user.getPassword());
		}
	}
}
```
### 3.测试
&emsp;&emsp;启动Zookeeper、Tomcat，依次启动SpringBootDubboServer、SpringBootDubboClient两个项目，SpringBootDubboClient端控制台会输出如下：
```bash
DubboClient启动成功。。。
用户名: username0，密码: password0
用户名: username1，密码: password1
用户名: username2，密码: password2
用户名: username3，密码: password3
用户名: username4，密码: password4
用户名: username5，密码: password5
用户名: username6，密码: password6
用户名: username7，密码: password7
用户名: username8，密码: password8
用户名: username9，密码: password9
用户名: username10，密码: password10
用户名: username11，密码: password11
用户名: username12，密码: password12
用户名: username13，密码: password13
用户名: username14，密码: password14
用户名: username15，密码: password15
用户名: username16，密码: password16
用户名: username17，密码: password17
用户名: username18，密码: password18
用户名: username19，密码: password19
```
同时Dubbo管理界面可查看到如下信息：
![](https://gitee.com/butalways1121/img-Blog/raw/master/111.png)
***
至此，完成了SpringBoot与Dubbo简单的整合。
