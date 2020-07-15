# SonarQube服务搭建

### 一 、 先决条件和概述

#### JDK 

| Java       | Server | Scanners |
| :--------- | :----- | :------- |
| Oracle JRE | 11     | ≥8       |
| OpenJDK    | 11     | ≥8       |

#### 数据库

| 数据库     | 版本 | 备注          |
| ---------- | ---- | ------------- |
| PostgreSQL | ≥9.3 | UTF-8 charset |
| Oracle     | ≥11g | UTF-8 charset |
| MSSQL      | ≥14  |               |



#### 硬件要求

实例至少需要2GB的RAM才能有效运行，而OS则需要1GB的可用RAM。

企业建议配置： 8C16G

Linux配置要求：

- `vm.max_map_count` 大于或等于524288
- `fs.file-max` 大于或等于131072
- 运行SonarQube的用户可以打开至少131072个文件描述符
- 运行SonarQube的用户可以打开至少8192个线程

检查方法

```bash
sysctl vm.max_map_count
sysctl fs.file-max
ulimit -n
ulimit -u
```



内核支持seccomp过滤器

默认情况下，Elasticsearch使用[seccomp filter](https://www.kernel.org/doc/Documentation/prctl/seccomp_filter.txt)。

```bash
$ grep SECCOMP /boot/config-$(uname -r)
```



字体

- [Fontconfig](https://en.wikipedia.org/wiki/Fontconfig)已安装在托管SonarQube的服务器上

- SonarQube服务器上安装了[FreeType](https://www.freetype.org/)字体包。可用的确切软件包会因分发而异，但是常用的软件包是`libfreetype6`

  

### 二、 在Docker上安装

#### 2.1 配置系统参数

修改/etc/sysctl.d/99-sonarqube.conf

```bash
vm.max_map_count=534288
fs.file-max=141072
```

让其生效

```bash
sysctl -p /etc/sysctl.d/99-sonarqube.conf
```



#### 2.2 编写docker-compose

/docker-compose/sonarqube/docker-compose.yml

参考： https://github.com/SonarSource/docker-sonarqube

```yam
version: "3"

services:
  sonarqube:
    image: sonarqube:8.2-community
    depends_on:
      - db
    ports:
      - "9000:9000"
    networks:
      - sonarnet
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
      - sonarqube_temp:/opt/sonarqube/temp
  db:
    image: postgres
    networks:
      - sonarnet
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
    volumes:
      - postgresql:/var/lib/postgresql
      # This needs explicit mapping due to https://github.com/docker-library/postgres/blob/4e48e3228a30763913ece952c611e5e9b95c8759/Dockerfile.template#L52
      - postgresql_data:/var/lib/postgresql/data

networks:
  sonarnet:
    driver: bridge

volumes:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
  sonarqube_temp:
  postgresql:
  postgresql_data:
```



#### 2.3 启动

```bash
cd /docker-compose/sonarqube
docker-compose up
```

访问：http://192.168.0.80:9000/

初始账户和密码： admin/admin



### 三、 服务器配置

#### 3.1  安装中文插件

Administration --> Marketplace  

搜索**Chinese Pack**



#### 3.2 修改初始密码

配置 > 权限 > 用户

##### 选择admin用户，点击轮滑按钮，选择重置密码



#### 3.3 创建新用户并生成token

配置 > 权限 > 用户  > 创建用户

创建完成后点击令牌

并记住令牌



### 四、 Jenkins与SonarQube集成插件的安装与配置

#### 4.1 安装Jenkins的SonarScanner

1. 系统管理 > 插件管理  > 可选插件
2. 搜索SonarQube Scanner
3. 点击直接安装



#### 4.2 配置SonarQube服务器

Jenkins面板点击：**系统管理 > 系统配置 >**  **SonarQube servers**  **>**  **Add SonarQube **

填入

Name:             Frog

Server URL：  http://192.168.0.80:9000

Token:   注意新加token，选择类型为Sercet file



#### 4.3 安装SonarQube Scanner

```bash
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.4.0.2170-linux.zip
unzip sonar-scanner-cli-4.4.0.2170-linux.zip
```



#### 4.4 配置SonarQube Scanner

编辑/opt/sonar-scanner-4.4.0.2170-linux/conf/sonar-scanner.properties

```
sonar.host.url=http://192.168.0.80:9000
sonar.sourceEncoding=UTF-8
```



#### 4.5 配置 SonarQube Scanner

进入Jenkins系统管理  > 全局工具配置

点击新增SonarQube Scanner,不勾选自动安装，

配置SonarQube Scanner

Name: sonar

**Sonar_Runner_Home:    /opt/sonar-scanner-4.4.0.2170-linux**

点击“保存”



#### 4.6 Jenkins配置检查任务

Jenkins > 项目 >  增加构建任务

选择**Execute SonarQube Scanner**



Task to run:  **scan**       # 即分析代码

Path to project properties:     这里可以指定一个 sonar-project.properties 文件，如果不指定的话会使用项目默认的 properties 文件

Analysis properties :  这里需要输入一些配置参数用来传递给 SonarQube，

这里的参数优先级高于 sonar-project.properties 文件里面的参数，

所以可以在这里来配置所有的参数以替代 sonar-project.properties 文件

```
sonar.projectKey=eploicy:print
sonar.projectName=Winglung Epolicy Print
sonar.projectVersion=1.0 
sonar.language=java
sonar.sourceEncoding=GBK
sonar.java.binaries=$WORKSPACE/webapps/WEB-INF/classes      
sonar.sources=$WORKSPACE/component/com/sinosoft/application
```

注解：

sonar.projectKey： 

sonar.java.binaries： 项目编译后二进制文件存放目录

sonar.sources： 需要分析的Java源码目录



### 五、  使用Ant和SonarQube集成

#### 5.1 下载SonarScanner for Ant

```bash
cd /opt
wget https://binaries.sonarsource.com/Distribution/sonarqube-ant-task/sonarqube-ant-task-2.7.0.1612.jar
```



#### 5.2 编写项目的build.xml

参考： https://github.com/SonarSource/sonar-scanning-examples

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project name="Simple Project analyzed with the SonarQube Scanner for Ant" default="all" basedir="." xmlns:sonar="antlib:org.sonar.ant">

	<!-- ========= Define the main properties of this project ========= -->
	<property name="src.dir" value="src" />
	<property name="build.dir" value="target" />
	<property name="classes.dir" value="${build.dir}/classes" />
	
	<!-- Define the SonarQube global properties (the most usual way is to pass these properties via the command line) -->
	<property name="sonar.host.url" value="http://192.168.0.80:9000" />

	<!-- Define the Sonar properties -->
	<property name="sonar.projectKey" value="org.sonarqube:sonarqube-scanner-ant" />
	<property name="sonar.projectName" value="Example of SonarQube Scanner for Ant Usage" />
	<property name="sonar.projectVersion" value="1.0" />
	<property name="sonar.sources" value="src" />
	<property name="sonar.binaries" value="target" />
	<property name="sonar.sourceEncoding" value="UTF-8" />
	
	<!-- ========= Define "regular" targets: clean, compile, ... ========= -->
	<target name="clean">
		<delete dir="${build.dir}" />
	</target>

	<target name="init">
		<mkdir dir="${build.dir}" />
		<mkdir dir="${classes.dir}" />
	</target>

	<target name="compile" depends="init">
		<javac srcdir="${src.dir}" destdir="${classes.dir}" fork="true" debug="true" includeAntRuntime="false" />
	</target>

	<!-- ========= Define SonarQube Scanner for Ant Target ========= -->
	<target name="sonar" depends="compile">
		<taskdef uri="antlib:org.sonar.ant" resource="org/sonar/ant/antlib.xml">
			<!-- Update the following line, or put the "sonar-ant-task-*.jar" file in your "$HOME/.ant/lib" folder -->
			<classpath path="/opt/sonarqube-ant-task-2.7.0.1612.jar" />
		</taskdef>
		
		<!-- Execute SonarQube Scanner for Ant Analysis -->
		<sonar:sonar />
	</target>

	<!-- ========= The main target "all" ========= -->
	<target name="all" depends="clean,compile,sonar" />

</project>

```



























