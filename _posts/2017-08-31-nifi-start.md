---
layout: post
title: NiFi引导性启动代码分析(1)
categories: nifi
description:
keywords: nifi
---
<strong>研究的源代码版本为1.3.0-SNAPSHOT,使用的环境为linux(fedora 4.7.2-201.fc24.x86_64)</strong>

在当前版本的代码中,启动被分成了3部分:

1. 启动脚本
1. bootstrap工程
1. nifi-framework-bundle工程

## 一.启动脚本 ##
&emsp;&emsp;启动脚本主要完成的功能是对整个启动工作中需要用到的classpath,NIFI启动参数(例如:NIFI_HOME,BOOTSTRAP_CONF等)基础运行参数做封装,另一层是对操作系统层面的启动做一次检测和隔离(例如:是否为cygwin环境,是否有运行权限等信息)...

### 1.脚本支持输入的参数 ###
```shell
case "$1" in
    install)
        install "$@"
        ;;
    start|stop|run|status|dump|env)
        main "$@"
        ;;
    restart)
        init
        run "stop"
        run "start"
        ;;
    *)
        echo "Usage nifi {start|stop|run|restart|status|dump|install}"
        ;;
esac
```
其中install是将执行自身程序的脚本加入到开机启动,

```shell
# Create the init script, overwriting anything currently present
cat <<SERVICEDESCRIPTOR > ${SVC_FILE}
#!/bin/sh
# Make use of the configured NIFI_HOME directory and pass service requests to the nifi.sh executable
NIFI_HOME=${NIFI_HOME}
bin_dir=\${NIFI_HOME}/bin
nifi_executable=\${bin_dir}/nifi.sh

\${nifi_executable} "\$@"
SERVICEDESCRIPTOR

    if [ ! -f "${SVC_FILE}" ]; then
        echo "Could not create service file ${SVC_FILE}"
        exit 1
    fi

    # Provide the user execute access on the file
    chmod u+x ${SVC_FILE}

    rm -f "/etc/rc2.d/S65${SVC_NAME}"
    ln -s "/etc/init.d/${SVC_NAME}" "/etc/rc2.d/S65${SVC_NAME}" || { echo "Could not create link /etc/rc2.d/S65${SVC_NAME}"; exit 1; }
    rm -f "/etc/rc2.d/K65${SVC_NAME}"
    ln -s "/etc/init.d/${SVC_NAME}" "/etc/rc2.d/K65${SVC_NAME}" || { echo "Could not create link /etc/rc2.d/K65${SVC_NAME}"; exit 1; }
    echo "Service ${SVC_NAME} installed"
}
```
其中init方法做了3个事情,分别为:


1. 验证系统是否为可支持的系统


1. 修改ulimit


1. 查找和验证java命令

start,stop,run,status,dump,env这几个启动命令会作为启动参数传递到bootstrap中执行
```
  run_nifi_cmd="'${JAVA}' ${DEBUG} -cp '${BOOTSTRAP_CLASSPATH}' -Xms12m -Xmx24m ${BOOTSTRAP_DIR_PARAMS} org.apache.nifi.bootstrap.RunNiFi $@"
```
<strong>其中的${DEBUG}是我方便远程调试加入的参数, DEBUG="-Xdebug -Xnoagent -Djava.compiler=NONE -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=2345";</strong>


## 二.bootstrap工程 ##

<strong>目前只研究启动流程,暂不对其他方法做研究</strong>

该工程做的事情也不是特别多,主要就是读取并解析bootstrap.conf中的配置项,然后封装成一个字符串:
```
 java -classpath /backup/nifi-1.3.0-SNAPSHOT/./conf:/backup/nifi-1.3.0-SNAPSHOT/./lib/slf4j-api-1.7.25.jar:/backup/nifi-1.3.0-SNAPSHOT/./lib/nifi-properties-1.3.0-SNAPSHOT.jar:/backup/nifi-1.3.0-SNAPSHOT/./lib/jcl-over-slf4j-1.7.25.jar:/backup/nifi-1.3.0-SNAPSHOT/./lib/logback-core-1.2.3.jar:/backup/nifi-1.3.0-SNAPSHOT/./lib/jetty-schemas-3.1.jar:/backup/nifi-1.3.0-SNAPSHOT/./lib/nifi-api-1.3.0-SNAPSHOT.jar:/backup/nifi-1.3.0-SNAPSHOT/./lib/nifi-nar-utils-1.3.0-SNAPSHOT.jar:/backup/nifi-1.3.0-SNAPSHOT/./lib/nifi-framework-api-1.3.0-SNAPSHOT.jar:/backup/nifi-1.3.0-SNAPSHOT/./lib/log4j-over-slf4j-1.7.25.jar:/backup/nifi-1.3.0-SNAPSHOT/./lib/nifi-runtime-1.3.0-SNAPSHOT.jar:/backup/nifi-1.3.0-SNAPSHOT/./lib/jul-to-slf4j-1.7.25.jar:/backup/nifi-1.3.0-SNAPSHOT/./lib/javax.servlet-api-3.1.0.jar:/backup/nifi-1.3.0-SNAPSHOT/./lib/logback-classic-1.2.3.jar -Dorg.apache.jasper.compiler.disablejsr199=true -Xmx512m -Xms512m -Djava.security.egd=file:/dev/urandom -Dsun.net.http.allowRestrictedHeaders=true -Djava.net.preferIPv4Stack=true -Djava.awt.headless=true -XX:+UseG1GC -Djava.protocol.handler.pkgs=sun.net.www.protocol -Dnifi.pro org.apache.nifi.NiFi
```
 用于启动org.apache.nifi.NiFi这个类,该类属于nifi-framework-bundle工程,是插件的核心框架.
```java
Process process = builder.start();
handleLogging(process);
Long pid = OSUtils.getProcessId(process, cmdLogger);
```
启动命令的字符串拼凑完成后,使用了一个ProcessBuilder去创建了一个操作系统的进程,然后将pid记录了下来,然后干了三件事情:

1. 为当前bootstrap虚拟机创建shutdownhook,也就是bootstrap虚拟机面临shutdown之前,会向org.apache.nifi.NiFi的虚拟机不断发送关闭指令,直到pid真正的不存在
```java
 final Socket socket = new Socket("localhost", ccPort);
 final OutputStream out = socket.getOutputStream();
 out.write(("SHUTDOWN " + secretKey + "\n").getBytes(StandardCharsets.UTF_8));
 out.flush();
```
1. 不断检测org.apache.nifi.NiFi的虚拟机是否关闭,如果发生关闭,不断重启该虚拟机.
1. 声明了一个NotificationServiceManager,对于当前的状态进行一个汇报,可以配置的选项如下,不做仔细研究,有兴趣的可以关注.
```xml
	<services>      
	<!--
		 <service>
			<id>email-notification</id>
			<class>org.apache.nifi.bootstrap.notification.email.EmailNotificationService</class>
			<property name="SMTP Hostname"></property>
			<property name="SMTP Port"></property>
			<property name="SMTP Username"></property>
			<property name="SMTP Password"></property>
			<property name="SMTP TLS"></property>
			<property name="From"></property>
			<property name="To"></property>
		 </service>
	-->
	<!--
		 <service>
			<id>http-notification</id>
			<class>org.apache.nifi.bootstrap.notification.http.HttpNotificationService</class>
			<property name="URL"></property>
		 </service>
	-->
	</services>
```

&emsp;&emsp;至于为什么要如此繁琐的的去启动一个软件,直接运行org.apache.nifi.NiFi就好了?可能作为作者而言,软件的层次需要清晰,架构的隔离上要严谨,
从shell上隔离系统的启动和检查,从bootstrap上隔离对插件以及jvm的监控和资源限制,真正到达org.apache.nifi.NiFi类后,再不需要关注任何启动的事情,专心致志的干启动插件的任务就可以了???? 

至此,引导性的启动已经分析完成,下次进入到插件机制的启动中.