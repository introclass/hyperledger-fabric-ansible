---
layout: default
title:  README
author: 李佶澳
createdate: 2018/07/18 19:00:00
changedate: 2018/07/26 18:55:33

---

## 支持版本

已经支持Fabric 1.2.x、Fabric 1.1.x，到同名分支中获取对应的代码。

Master分支用于等待下一个版本。

## 说明

这是网易云课堂“[IT技术快速入门学院][1]”第二门课《[HyperLedger Fabric进阶实战课][2]》第三章节使用的素材。

这其实已经是一套非常实用的Ansible部署脚本了，完全可以应用于生产：[视频演示讲解][2]。

[![video](http://www.lijiaocn.com/img/player.png)](https://study.163.com/course/introduction.htm?courseId=1005326005#/courseDetail?tab=1)

要获得更多的内容，可以关注：

        微信公众号： “我的网课”，(关注后可以获得我微信)
        QQ交流群： 576555864

如果视频中有讲解不到位或需要订正的地方，可以加入知识星球“区块链实践分享”，（二维码在最后）

## 直接部署Fabric-1.2.x

直接部署过程与分支Fabric-1.1.x的部署过程类似，只是将程序文件换成了1.2.0版本。

	 Version: 1.2.0
	 Commit SHA: cae2ad4
	 Go version: go1.10
	 OS/Arch: linux/amd64
	 Experimental features: false
	 Chaincode:
	  Base Image Version: 0.4.10
	  Base Docker Namespace: hyperledger
	  Base Docker Label: org.hyperledger.fabric
	  Docker Namespace: hyperledger

	$(DOCKER_NS)/fabric-ccenv:latest
	$(BASE_DOCKER_NS)/fabric-baseos:$(ARCH)-$(BASE_VERSION)
	$(DOCKER_NS)/fabric-javaenv:$(ARCH)-1.1.0
	$(BASE_DOCKER_NS)/fabric-baseimage:$(ARCH)-$(BASE_VERSION)

Fabric1.2.x默认使用下面的镜像，最好在peer上提前下载好：

	docker pull hyperledger/fabric-ccenv:latest
	docker pull hyperledger/fabric-baseos:amd64-0.4.10
	docker pull hyperledger/fabric-javaenv:x86_64-1.1.0     //for java
	docker pull hyperledger/fabric-baseimage:amd64-0.4.10   //for node.js

### 目标

在192.168.88.10、192.168.88.11、192.168.88.12上部署一个有两个组织三个Peer组成的联盟。

联盟的二级域名为： example.com。

组织一的域名为： member1.example.com 

组织二的域名为： member2.example.com

组织一中部署了一个Orderer和两个Peer，域名和IP分别为：

	orderer0.member1.example.com  192.168.88.10
	peer0.member1.example.com     192.168.88.10
	peer1.member1.example.com     192.168.88.11

组织二没有部署Orderer参与共识，只部署一个Peer：

	peer0.member2.example.com     192.168.88.12

共识算法是solo，如果要切换为其它共识算法，例如kafka，需要另外部署kafka，并修改配置文件。

	$ tree inventories/example.com/
	inventories/example.com/
	├── configtx.yaml            //配置参与组织，用于生成创世块和Channel文件
	├── crypto-config.yaml       //配置参与组织，用于生成证书
	├── etc_hosts                //域名与机器IP的对应
	├── group_vars
	│   └── all                  //设置变量
	├── host_vars
	└── hosts                    //目标机器

### 准备

0 将要部署到目标环境中的二进制文件复制到output/example.com/bin/目录中

	mkdir -p output/example.com/
	cd output/example.com/
	wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.2.0/hyperledger-fabric-linux-amd64-1.2.0.tar.gz
	wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.2.0/hyperledger-fabric-linux-amd64-1.2.0.tar.gz.md5
	tar -xvf hyperledger-fabric-linux-amd64-1.2.0.tar.gz
	cd ../../

1 在inventories/example.com中创建配置文件，以及ansible需要的hosts文件:

	configtx.yaml
	crypto-config.yaml
	etc_hosts
	group_vars/all
	hosts

2 准备在运行ansible的机器中使用fabric命令。

`prepare.sh`会使用hyperledger fabric的命令，需要把在本地运行的fabric命令放到`output/bin`目录中。

我是在mac上执行ansible的，下载的是darwin版本的fabric：

	mkdir -p output/bin
	cd output/bin
	wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/darwin-amd64-1.2.0/hyperledger-fabric-darwin-amd64-1.2.0.tar.gz
	wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/darwin-amd64-1.2.0/hyperledger-fabric-darwin-amd64-1.2.0.tar.gz.md5
	tar -xvf hyperledger-fabric-darwin-amd64-1.2.0.tar.gz
	cd ../../

3 运行perpare.sh生成证书，以及创世块(可以根据需要修改脚本)：

	./prepare.sh example

>每个部署环境分别在output和inventories中有一个自己的目录，要增加新部署环境除了在output和inventories中准备目录和文件，您还可能需要根据自己的需要在prepare.sh中添加为新的环境生成证书和其它文件的命令。

4 准备Docker安装文件

	cd roles/prepare/files/
	wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-18.03.1.ce-1.el7.centos.x86_64.rpm

### 部署

1 初始化目标机器

	export ANSIBLE_HOST_KEY_CHECKING=False
	ansible-playbook -k -i inventories/example.com/hosts -u root deploy_prepare.yml

2 检测证书设置是否成功

	ansible -i inventories/example.com/hosts -u root  all  -m command -a "pwd"

3 如果域名没有绑定IP，修改每台机器的/etc/hosts，（会替换整个文件）：

	ansible -i inventories/example.com/hosts -u root  all  -m copy -a "src=./inventories/example.com/etc_hosts dest=/etc/hosts"

4 部署节点

	ansible-playbook -i inventories/example.com/hosts -u root deploy_nodes.yml

5 部署客户端

	ansible-playbook -i inventories/example.com/hosts -u root deploy_cli.yml

### Fabric初始化

1 进入member1的管理员目录，对peer0.member1.example.com进行操作：

	cd /opt/app/fabric/cli/user/member1.example.com/Admin-peer0.member1.example.com/
	
	//创建channel，channel只需要创建一次
	./0_create_channel.sh
	
	//加入channel
	./1_join_channel.sh
	
	//设置锚点Peer：
	./2_set_anchor_peer.sh

2 进入member1的管理员目录，对peer1.member1.example.com进行操作：

	cd /opt/app/fabric/cli/user/member1.example.com/Admin-peer1.member1.example.com
	./1_join_channel.sh

3 进入member2的管理员目录，对peer0.member1.example.com进行操作：

	cd /opt/app/fabric/cli/user/member2.example.com/Admin-peer0.member2.example.com
	
	//加入channel
	./1_join_channel.sh
	
	//设置锚点Peer：
	./2_set_anchor_peer.sh

### 部署合约

1 进入member1的管理员目录，对peer0.member1.example.com进行操作：

	cd /opt/app/fabric/cli/user/member1.example.com/Admin-peer0.member1.example.com/
	
	//先获取合约代码，可能会比较慢，拉取代码比较耗时
	go get github.com/lijiaocn/fabric-chaincode-example/demo
	
	//安装合约
	./3_install_chaincode.sh
	
	//查看已经安装的合约
	./peer.sh chaincode list --installed
	
	//合约实例化，只需要实例化一次
	./4_instantiate_chaincode.sh

2 在其它Peer上部署合约

	//peer1.member1.example.com
	//先获取合约代码，可能会比较慢，拉取代码比较耗时
	go get github.com/lijiaocn/fabric-chaincode-example/demo
	
	cd /opt/app/fabric/cli/user/member1.example.com/Admin-peer1.member1.example.com/
	./3_install_chaincode.sh
	
	//peer0.member2.example.com
	//先获取合约代码，可能会比较慢，拉取代码比较耗时
	go get github.com/lijiaocn/fabric-chaincode-example/demo
	
	cd /opt/app/fabric/cli/user/member2.example.com/Admin-peer0.member2.example.com/
	./3_install_chaincode.sh

>同一个合约，只需要在任意一个Peer上实例化一次。

3 调用合约，写数据

	./6_invoke_chaincode.sh

4 调用合约，查数据

	./5_query_chaincode.sh

### 管理操作

1 启动链：

	ansible-playbook -i inventories/example.com/hosts -u root playbooks/manage_start.yml

2 停止链：

	ansible-playbook -i inventories/example.com/hosts -u root playbooks/manage_stop.yml

3 清空链上所有数据：

	ansible-playbook -i inventories/example.com/hosts -u root playbooks/manage_rebuild.yml

4 销毁链：

	ansible-playbook -i inventories/example.com/hosts -u root playbooks/manage_destroy.yml


## 从 Fabric 1.1 升级到 Fabric 1.2

**重要**: 升级要在部署Fabric 1.1时使用的`hyperledger-fabric-ansible`目录中进行操作。

备份上一个版本的二进制文件，注意只备份bin和config：

	cd output/example.com
	mv bin bin-1.1.0
	mv config config-1.1.0

**注意1**：不要改动output/example.com中的`crypto-config`，这个目录中存放的是证书，在升级时不应当被更新！

下载1.2版本的文件:

	wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.2.0/hyperledger-fabric-linux-amd64-1.2.0.tar.gz
	wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.2.0/hyperledger-fabric-linux-amd64-1.2.0.tar.gz.md5
	tar -xvf hyperledger-fabric-linux-amd64-1.2.0.tar.gz

对比config和config-1.1.0中的文件，看一下1.2.0版本的配置文件中引入了哪些新的配置。

将原先的配置文件备份：

	mv ../../roles/peer/templates/core.yaml.j2 config-1.1.0/
	mv ../../roles/orderer/templates/orderer.yaml.j2 config-1.1.0/
	mv ../../roles/cli/templates/core.yaml.j2 config-1.1.0/

然后在config中准备最新的配置模版：

	cd config
	cp core.yaml core.server.yaml.j2
	cp core.yaml core.client.yaml.j2
	cp orderer.yaml  orderer.yaml.j2

编辑core.yaml.j2和orderer.yaml.j2之后，将其复制到对应的目录：

	cp `pwd`/config/orderer.yaml.j2       ../../roles/orderer/templates/orderer.yaml.j2
	cp `pwd`/config/core.server.yaml.j2   ../../roles/peer/templates/core.yaml.j2
	cp `pwd`/config/core.client.yaml.j2   ../../roles/cli/templates/core.yaml.j2


**注意2**：下面是直接关停所有节点，然后用anbile一次替换所有节点上的程序文件，生产环境中注意要逐台升级，并做好备份！

关停节点：

	ansible-playbook -i inventories/example.com/hosts -u root ./playbooks/manage_stop.yml

`Ansible脚本能确保只更新发生了变化的文件，应当只有程序文件或者更新后的配置文件被更新`

更新所有机器上的程序文件：

	ansible-playbook -i inventories/example.com/hosts -u root deploy_nodes.yml

更新cli中的程序文件：

	ansible-playbook -i inventories/example.com/hosts -u root deploy_cli.yml

验证:

	$ cd /opt/app/fabric/cli/user/member1.example.com/Admin-peer0.member1.example.com
	$ ./peer.sh node status
	status:STARTED

原先的数据和合约依旧可以使用：

	$ ./5_query_chaincode.sh
	key1value

## 联系

要获得更多的内容，可以关注：

        微信公众号： “我的网课”，(关注后可以获得我微信)
        QQ交流群： 576555864

如果视频中有讲解不到位或需要订正的地方，可以加入：

![知识星球区块链实践分享](http://www.lijiaocn.com/img/xiaomiquan-blockchain.jpg)

## 参考

1. [网易云课堂，IT技术快速入门学院][1]
2. [HyperLedger Fabric进阶实战课][2]
3. [超级账本&区块链实战文档][3]
4. [HyperLedger Fabric原版文档中文批注][4]

[1]: https://study.163.com/provider/400000000376006/course.htm?share=2&shareId=400000000376006 "IT技术快速入门学院" 
[2]: https://study.163.com/course/courseMain.htm?courseId=1005359012&share=2&shareId=400000000376006 "HyperLedger Fabric进阶实战课"
[3]: http://www.lijiaocn.com/tags/blockchain.html "超级账本&区块链实战文档"
[4]: http://fabric.lijiaocn.com "HyperLedger Fabric原版文档中文批注"
