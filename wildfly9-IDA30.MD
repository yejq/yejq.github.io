# wildfly-9.0.2.Final部署IDA30说明 #

下载地址：[http://download.jboss.org/wildfly/9.0.2.Final/wildfly-9.0.2.Final.zip](http://download.jboss.org/wildfly/9.0.2.Final/wildfly-9.0.2.Final.zip)

假设wildfly的目录在：/app/wildfly-9.0.2.Final

## 1、standalone.xml配置 ##

/app/wildfly-9.0.2.Final/standalone/configuration/standalone.xml，配置内容参考附件的standalone.xml。

### 日志配置 ###

	<subsystem xmlns="urn:jboss:domain:logging:3.0">

### 数据源配置 ###
	
	<subsystem xmlns="urn:jboss:domain:datasources:3.0">

	<subsystem xmlns="urn:jboss:domain:ee:3.0">里边的<default-bindings>修改datasource为java:/AppDB
	
### 服务器改成支持外网访问 ###

	<interfaces>增加：
	<interface name="any">
    	<any-address/>
    </interface>

	<socket-binding-group name="standard-sockets" 
	修改default-interface="any"

### 服务器http端口配置 ###
	
	 <socket-binding-group name="standard-sockets"
		<socket-binding name="http" port="${jboss.http.port:8080}"/>
	这里配置的是8080端口。
	

## 2、模块配置 ##

### oracle驱动 ###

拷贝附件的oracle目录到：/app/wildfly-9.0.2.Final/modules/system/layers/base/com
	
### IDA30配置文件 ###

附件目录里边regaltec\configuration\main是配置好的样例；需要删除除module.xml以外的其他文件，复制IDA30项目里边\WebRoot\WEB-INF\classes里的文件到这个目录；
	
	
**注意点：以后修改了\WebRoot\WEB-INF\classes里的配置文件需要覆盖到regaltec\configuration\main里边；**

### JDK配置 ###
修改/app/wildfly-9.0.2.Final/modules/system/layers/base/sun/jdk/main/module.xml，增加：

	<path name="sun/text"/>
	<path name="sun/net"/>
	<path name="sun/net/ftp"/>

## 3、JDK配置 ##
wildfly-9.0.2.Final使用jdk1.7+，假设jdk安装目录是/usr/java/jdk1.7.0_79，需要修改/usr/java/jdk1.7.0_79/jre/lib/rt.jar。

由于IDA30使用了sun/net/ftp里边的类，但这些类只有在jdk1.6之前有，jdk1.7已经重构了这个包。

处理步骤：

1、复制备份rt.jar：

	cp /usr/java/jdk1.7.0_79/jre/lib/rt.jar /usr/java/jdk1.7.0_79/jre/lib/rt.jar.bak
	
2、 清空rt.jar里边目录sun/net/ftp内的类文件，复制jdk1.6版本的sun/net/ftp里边的类文件到jdk1.7这个目录内；

## 4、应用发布 ##

### 指定JDK配置（可选） ###
修改：/app/wildfly-9.0.2.Final/bin/standalone.conf

	JAVA_HOME="/usr/java/jdk1.7.0_79"

### 修改启动脚本 ###
修改：/app/wildfly-9.0.2.Final/bin/run.sh

	sh standalone.sh -Djboss.server.base.dir=./../standalone -Djboss.node.name=node8080 &

### 应用发布 ###
拷贝IDA30.war到/app/wildfly-9.0.2.Final/standalone/deployments

执行启动脚本：
	
	cd /app/wildfly-9.0.2.Final/bin
	./run.sh

**注意：应用发布可以不删除缓存，可以不关闭应用的情况下拷贝包，直接拷贝IDA30.war到deployments目录覆盖，拷贝完之后，拷贝和重启过程中会生成IDA30.war.isdeploying，这个文件被删除之后，生成IDA30.war.deployed，表示发布完成，不过一般由于应用过于繁忙，自动重启不了，要手工重启；**

### 缓存目录 ###
	/app/wildfly-9.0.2.Final/standalone/tmp/vfs/temp
	案例：/app/wildfly-9.0.2.Final/standalone/tmp/vfs/temp/temp883820ea3144adf1/content-64ce5b6b76e2f9ad

## 5、standalone(独立运行模式)和domain(域模式) ##

jboss提供了二种运行模式：standalone(独立运行模式)、domain(域模式)，日常开发中，使用standalone模式足已；但生产部署时，一个app，往往是部署在jboss集群环境中的，如果所有jboss server均采用standalone模式，会给运维带来极大的工作量，需要每台jboss server上逐一部署/更新，显然不适合。

domain模式正是为了解决这一问题，该模式下，所有jboss server可以划分成不同的group(注：这里的jboss server并不一定要对应某台物理机或虚拟机，一个os上，可以同时run多个jboss server实例，所以本文中的jboss server均指某个运行中的jboss server instance)，每个group中可以包含多个jboss server，所有这些jboss server中，可以指定一台做为域控制器(domain controller)，俗称master server，其它jboss server均为Home Controller(俗称slave server)。

master上可以控制所有jboss server，并监控其运行情况，部署应用时，一个war包，只需要部署到group上，该group中的所有jboss server即会同步自动部署。

两种模式都可以做负载均衡，需要使用ha-profile；负载方面原生支持Apache Httpd，在HA方面默认也是使用Apache Httpd。对Apache不感冒，也可以用nginx，其中一个方案是Session方面使用nginx-sticky-module模块来做处理，而网络检测使用nginx\_upstream\_check\_module，更多高可用方面的信息，可以查看[https://yejq.github.io/wildfly9/High_Availability_Guide.html](https://yejq.github.io/wildfly9/High_Availability_Guide.html "HighAvailabilityGuide")。


## 6、standalone模式配置多实例 ##
比如要配置两个实例，一个端口是8080，一个端口是8180；完成好单个实例的配置之后，直接复制一份即可。

配置步骤：

1、修改standalone目录为standalone8080，执行命令：
	
	mv /app/wildfly-9.0.2.Final/standalone /app/wildfly-9.0.2.Final/standalone8080

2、复制standalone8080为standalone8180，执行命令：

	cp /app/wildfly-9.0.2.Final/standalone8080 /app/wildfly-9.0.2.Final/standalone8180

3、修改启动脚本/app/wildfly-9.0.2.Final/bin/run.sh：

	sh standalone.sh -Djboss.server.base.dir=./../standalone8080 -Djboss.node.name=node8080 &
	sleep 5
	sh standalone.sh -Djboss.server.base.dir=./../standalone8180 -Djboss.node.name=node8180 -Djboss.socket.binding.port-offset=100 &

## 7、domain模式 ##

部署一台master server，其余都配成slave server，先启动master server，再启动slave server，以后server就可以不用启动和停止了，配置、应用启动和停止、应用发布等几乎所有操作都在管理控制台进行。登陆 http://ip:9990/ 操作即可；



待续。


> enjoy it
