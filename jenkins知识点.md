一、概述
    jenkins是什么？
    一个CI/CD软件，用于自动化各种任务，包括构建，测试和部署软件，
    jenkins的前身是hudson。
CI:continuous  INTEGRATION  可以持续化集成
CD:  continuous   delivery  、continuous  deployment 可持续化交互/可持续化发布









二、安装（官网地址 https://jenkins.io/）
1、Jenkins有两种版本，一种安装版，一种war包（推荐）。各种版本的安装版的安装方法： https://jenkins.io/zh/doc/book/installing/
2、三种启动方式：
①、宿主机上的tomcat启动，下载war包，直接放在tomcat的webapp是目录，启动tomcat就可以使用了，访问：http://localhost:8080/jenkins/（首选）
②、docker上启动。（不推荐，贼鸡儿麻烦；要在容器中安装jdk、maven、git，还有相应的配置。仓库配置，文件交互比较麻烦，总之不要在docker上使用jenkins）
③、java -jar 命令 会默认启动一个tomcat。还不如直接第一种方式。
ps：
①第一次启动，控制台会显示默认密码。










三、环境准备
在一台主机上准备两个tomcat，需要注意两个tomcat的启动端口和关闭端口不能重复，其中一个tomcat-jenkins运行了jenkins.war，另一个tomcat-app运行项目war包。









四、jenkins全局配置
1、本机安装git、jdk（1.8）、maven（>=3.3.9）
2、在jenkins控制台上安装插件：Maven Integration集成(可以创建maven类型的任务)、Deploy to container动态部署










五、构建一个动态部署应用的任务（必须将项目打成war才能再一个tomcat动态部署项目，jar包不行）
创建任务并配置
1、源码管理（从git上拉取项目源代码）
2、构建触发器 （检查git上是否有代码更新，重新构建项目）
3、构建后操作（把项目部署到另一个tomcat下的webapps下）
    该tomcat-app需要以下配置：
修改 Tomcat目录/webapps/manager/META-INF/context.xml
注释调用以下内容:
<Context antiResourceLocking="false" privileged="true" >
  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />定义了是哪些ip可以访问，因此需要关闭
修改tomcat目录/conf/tomcat-user.xml
添加一个用户
<role rolename="manager-script">
<role rolename="admin-script">
<role rolename="manager-gui">
<role rolename="admin-gui">
<user username="tomcat" password="123456" roles="manager-script,admin-script,admin-gui,manager-gui">


ps：
1、jenkins工作空间(任务对应数据、项目工程)
/Users/mac/.jenkins/workspace
2、定时器语法（定时构建 尽量是大于5分钟每次，实验时不要设置成H/1 * * * *  这个值有bug）
每隔5分钟构建一次    H/5 * * * *
每两小时构建一次    H H/2 * * *
每天中午下班前定时构建一次   0 12 * * *
每天下午下班前定时构建一次  0 18 * * *

3、轮询构建 在定时构建在基础上，在构建之前去git上拉取最新代码。如果git上没有更新则不会执行构建 （推荐）
4、修改了配置需要点击立即构建才能生效
5、这种动态部署自能开发者在自己的开发机上自己使用，毕竟tomcat总是不重启还要动态的部署war包，总是有出错的的时候.JVM总是内存溢出等各种原因，导致项目无法正常部署。











六、构建使用shell脚本的方式部署应用的 任务
相比第三点、不重启tomcat，第四点通过shell脚本重启tomcat，保证应用能够正常部署。
创建任务并配置
1、源码管理（从git上拉取项目源代码）
2、构建触发器 （检查git上是否有代码更新，重新构建项目）
3、Post Steps >执行shell
    shell流程：
    ①关闭原先的tomcat进程
    ②把war包复制到tomcat webapps下
    ③启动tomcat
    shell脚本内容：
------------------shell start-----------------------------------------------------    
#!/bin/sh
#kill tomcat pid

#这句尤为重要:因为jenkins执行脚本结束后，就认为任务结束了，但是启动的相关子程序依然在运行（由于Jenkins认为任务已经结束了，就结束该构建相关的子进程）
#所以需要配置一个环境变量，不然启动的Tomcat会被关掉
export BUILD_ID=apache-tomcat-app_build_id

# 1.关闭tomcat
pidlist=`ps -ef|grep tomcat-app|grep -v "grep"|awk '{print $2}'`
function stop(){
if [ "$pidlist" == "" ]
  then
    echo "----tomcat 已经关闭----"
    
 else
    echo "tomcat进程号 :$pidlist"
    kill -9 $pidlist
    echo "KILL $pidlist:"
fi
}

stop
pidlist2=`ps -ef|grep tomcat-app|grep -v "grep"|awk '{print $2}'`
if [ "$pidlist2" == "" ]
    then 
       echo "----关闭tomcat成功----"
else
    echo "----关闭tomcat失败----"
fi

# 2.移除原来tomcat中webapps中的项目文件夹
rm -rf /usr/local/tomcat-app/webapps/ROOT*
# 3.复制jenkins生成的war包到tomcat中webapps中
cp -r /var/root/.jenkins/workspace/dockertest/target/dockertest.war  /usr/local/tomcat-app/webapps
sleep 3s
# 4.修改war包的名称
mv /usr/local/tomcat-app/webapps/dockertest.war  /usr/local/tomcat-app/webapps/ROOT.war
# 5.启动tomcat
cd /usr/local/tomcat-app/bin
./startup.sh
------------------shell end-----------------------------------------------------  

ps -ef|grep apache-tomcat-app|grep -v "grep"|awk '{print $2}'
1、ps -ef 显示所有的进程，其中后面的e是显示结果的意思，f是显示完整格式，具体可见ps --help all
2、ps -ef|grep grep apache-tomcat-app作用是把包括grep apache-tomcat-app这个关键字的进程都显示出来
3、同时 ps -ef|grep grep apache-tomcat-app会把grep cpu的进程也统计进来，因此用ps -ef|grep grep apache-tomcat-app|grep -v grep去除grep进程
4、最后，只包含apache-tomcat-app关键字的进程筛选结果作为输入给awk '{print $2}'，这个部分的作用是提取输入的第二列，而第二列正是进程的PID

远程部署：安装ssh插件












七、DevOps 开发和运维的组合
参考图片
https://develop.aliyun.com/devops









八、集群部署的思路
①修改shell脚本
②多创建几个任务
部署项目，变数比较多，项目规模的复杂度不一。更多的通过修改shell脚本的方式来部署应用。
规模越大越需要配置中心。
shell脚本能搞定的功能jenkins一定能搞定，因此再完成某个功能之前看看有没有合适的插件

九、jenkins可以集成自动化测试，
略


ps:任意语言的项目都可以使用jenkins部署项目。