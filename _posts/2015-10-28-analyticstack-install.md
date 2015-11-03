---
layout: post
title: analyticstack版本的edX开发环境安装
category: edx
excerpt: "首先我们在mirrors.edxstack.org下载所需要的analyticstack.box文件,在某个盘里新建目录analyticstack，然后用cmd，进入到你的对应的目录位置..."
modified: 2015-11-03
tags: [edx,analyticstack]
comments: true
pinned: true
---
#准备工作
宿主机环境
1. Ubuntu
2. win7
3. win7下的虚拟机ubuntu

#安装
　　首先如果你是第三种的话，一般的机器是不支持虚拟机再安装虚拟机的（CPU架构问题，就算可以对cpu要求太高了），所以要么换系统要么装个双系统。下面来说说第二种是怎么安装。

　　我们需要[vagrant](https://www.vagrantup.com/)，[virtualbox](https://www.virtualbox.org/)，vagrant我们需要需改它box存放的根目录，不然C盘空间会成为安装的障碍。copy ~/.vagrant.d/下面的目录到新目录
cp ~/.vagrant.d/   /path/to/vagrant_home/
设置环境变量
grep 'VAGR' ~/.bashrc
export VAGRANT_HOME='/path/to/vagrant_home'
（当然以上是Ubuntu下的方法，在win7下你只需要复制.vagrant.d文件到指定地方，然后添加环境变量VAGRANT_HOME）  
　　还有[xshell](http://www.netsarang.com/products/xsh_overview.html)这三个工具，怎么安装成功百度即可，没有难度，但是版本必须大于edx要求的版本，一般最新版即可[analytics-devstack版本要求](http://edx.readthedocs.org/projects/edx-installing-configuring-and-running/en/latest/devstack/analytics_devstack.html#using-the-analytics-devstack)，在这个链接里面写出来了怎么来安装，但是由于国内的网络环境，我们经常是被卡在墙外的，所以必须另辟蹊径。

　　首先我们在mirrors.edxstack.org下载所需要的analyticstack.box文件,在某个盘里新建目录analyticstack，然后用cmd，进入到你的对应的目录位置　　
```
f:(假设在f盘)  
cd analyticstack  
vagrant init(没有这一步会导致后面文件读写和解压失败),添加虚拟机文件  
vagrant box add analyticstack analyticstack.box（box文件不在当前目录vagrant box add analyticstack file:///e:\download\analyticstack.box），等待十几分钟的样子，如果成功了，继续下一步，  
vagrant init analyticstack 初始化虚拟机  
vagrant up 千万不能再第一次 vagrant up的时候中断，不然会造成一些其他的问题(我第一次就造成文件丢失)，如果顺利的话就会看到启动成功的界面，再  
vagrant ssh 开启ssh连接
```  
　　使用xshell新建链接，输入ip和端口号，一般是127.0.0.1和2222，每次vagrant ssh会有提示。然后提示输入用户名和密码，都是vagrant。  
![xshell](http://i.imgur.com/onten3M.png)

#启用edX Analytics Data API

```
sudo su analytics_api
~/venvs/analytics_api/bin/python ~/analytics_api/manage.py runserver 0.0.0.0:8100 --insecure
```

#运行edX Insights

```
sudo su insights 每次进入新的用户的时候需要先exit退出当前的用户登录一下，不然要输入密码  
~/venvs/insights/bin/python ~/edx_analytics_dashboard/manage.py switch display_verified_enrollment on --create
~/venvs/insights/bin/python ~/edx_analytics_dashboard/manage.py switch enable_course_api on --create
~/venvs/insights/bin/python ~/edx_analytics_dashboard/manage.py runserver 0.0.0.0:8110 --insecure
```
用浏览器打开：http://127.0.0.1:8110

#执行数据分析任务(edX Analytics Pipeline)
vagrant ssh进入devstack，注册用户，注册课程
开始学习，回答几道课程中的问题
退出devstack，回到宿主机，进入edx-analytics-pipeline目录
执行任务

1. 统计注册信息：
```
export WHEEL_URL=http://edx-wheelhouse.s3-website-us-east-1.amazonaws.com/Ubuntu/precise
remote-task --vagrant-path analyticstack--remote-name devstack --override-config ${PWD}/config/devstack.cfg --wheel-url   
$WHEEL_URL --wait \ ImportEnrollmentsIntoMysql --local-scheduler --interval-end $(date +%Y-%m-%d -d "tomorrow") --n-reduce-tasks 1
```
2. 统计答案分布：
```
export WHEEL_URL=http://edx-wheelhouse.s3-website-us-east-1.amazonaws.com/Ubuntu/precise
export UNIQUE_NAME=$(date +%Y-%m-%dT%H_%M_%SZ)  
remote-task --vagrant-path analyticstack--remote-name devstack --override-config ${PWD}/config/devstack.cfg --wheel-url 
$WHEEL_URL --wait \  `
AnswerDistributionWorkflow --local-scheduler   
--src hdfs://localhost:9000/data/ \  
--include 'tracking.log' \  
--dest hdfs://localhost:9000/edx-analytics-pipeline/output/answer_distribution_raw/$UNIQUE_NAME/data \  
--name $UNIQUE_NAME   
--output-root hdfs://localhost:9000/edx-analytics-pipeline/output/answer_distribution/ \  
--marker hdfs://localhost:9000/edx-analytics-pipeline/output/answer_distribution_raw/$UNIQUE_NAME/marker \  
--n-reduce-tasks 1
```
