---
layout: default
title:  README
author: 李佶澳
createdate: 2018/07/24 12:48:00
changedate: 2018/07/24 14:37:22

---

## 支持版本

Fabric-1.1.x，对其它版本的支持在与版本号同名的Branch中。

## 说明

这是网易云课堂“[IT技术快速入门学院][1]”的
第二门课《[HyperLedger Fabric进阶实战课][2]》第三章节使用的素材。

这其实已经是一套非常实用的Ansible部署脚本了，完全可以应用于生产：[视频演示讲解][2]。

要获得更多的内容，可以关注：

        微信公众号： “我的网课”，(关注后可以获得我微信)
        QQ交流群： 576555864

如果视频中有讲解不到位或需要订正的地方，可以加入知识星球“区块链实践分享”，（二维码在最后）

## 目标

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

## 准备

0 将要部署到目标环境中的二进制文件复制到output/example.com/bin/目录中

	mkdir -p output/example.com/
	cd output/example.com/
	wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.1.0/hyperledger-fabric-linux-amd64-1.1.0.tar.gz
	wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.1.0/hyperledger-fabric-linux-amd64-1.1.0.tar.gz.md5
	tar -xvf hyperledger-fabric-linux-amd64-1.1.0.tar.gz

使用yum安装Docker，可能会因为qiang的原因安装失败，需要提前下载rpm：

	mkdir -p roles/prepare/files/
	cd roles/prepare/files/
	wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-18.03.1.ce-1.el7.centos.x86_64.rpm

1 在inventories/example.com中创建配置文件，以及ansible需要的hosts文件:

	configtx.yaml
	crypto-config.yaml
	hosts

2 准备在运行ansible的机器使用fabric命令：

注意事项1：

>`prepare.sh`会使用hyperledger fabric的命令，需要把在本地运行的fabric命令放到`output/bin`目录中。

例如，我是在mac上执行ansible的，下载的是darwin版本的fabric：

	mkdir -p output/bin
	cd output/bin
	wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/darwin-amd64-1.1.0/hyperledger-fabric-darwin-amd64-1.1.0.tar.gz
	wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/darwin-amd64-1.1.0/hyperledger-fabric-darwin-amd64-1.1.0.tar.gz.md5
	tar -xvf hyperledger-fabric-darwin-amd64-1.1.0.tar.gz

3 运行perpare.sh生成证书，以及创世块(可以根据需要修改脚本)：

	./prepare.sh example

注意事项2：

>每个部署环境分别在output和inventories中有一个自己的目录，要增加新部署环境除了在output和inventories中准备目录和文件，您还可能需要根据自己的需要在prepare.sh中添加为新的环境生成证书和其它文件的命令。

## 部署

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

6 如果要在当前机器上，操作所有peer，可以在本地安装所有peer的客户端：

	sudo mkdir /opt/app
	chown lijiao /opt/app/
	ansible-playbook -i inventories/example.com/hosts -u root deploy_cli_local.yml
	//注意配置本地的host

## Fabric初始化

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

## 部署合约

在实例化合约的时候，会联网下载几个比较大的镜像（2～3G），会导致合约实例化等待非常长的时间。

可以从[素材下载][5]中下载已经打包好的docker镜像，将其在每个Peer上导入:

	tar -xvf docker-images.tar.gz
	cd docker-images
	./load.sh

或者提前到每个Peer上执行：

	docker pull hyperledger/fabric-javaenv:x86_64-1.1.0
	docker pull hyperledger/fabric-ccenv:x86_64-1.1.0

1 进入member1的管理员目录，对peer0.member1.example.com进行操作：

	cd /opt/app/fabric/cli/user/member1.example.com/Admin-peer0.member1.example.com/
	
	//先获取合约代码，可能会比较慢，拉取代码比较耗时
	go get github.com/lijiaocn/fabric-chaincode-example/demo
	
	//安装合约
	./3_install_chaincode.sh
	
	//查看已经安装的合约
	./peer.sh chaincode list --installed
	
	//合约实例化，只需要实例化一次，这个过程docker会拉取镜像，会比较慢。
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

## 管理操作

1 启动链：

	ansible-playbook -i inventories/example.com/hosts -u root playbooks/manage_start.yml

2 停止链：

	ansible-playbook -i inventories/example.com/hosts -u root playbooks/manage_stop.yml

3 清空链上所有数据：

	ansible-playbook -i inventories/example.com/hosts -u root playbooks/manage_rebuild.yml

4 销毁链：

	ansible-playbook -i inventories/example.com/hosts -u root playbooks/manage_destroy.yml

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
5. [素材下载][5]

[1]: https://study.163.com/provider/400000000376006/course.htm?share=2&shareId=400000000376006 "IT技术快速入门学院" 
[2]: https://study.163.com/course/courseMain.htm?courseId=1005359012&share=2&shareId=400000000376006 "HyperLedger Fabric进阶实战课"
[3]: http://www.lijiaocn.com/tags/blockchain.html "超级账本&区块链实战文档"
[4]: http://fabric.lijiaocn.com "HyperLedger Fabric原版文档中文批注"
[5]: https://pan.baidu.com/s/1XgPqCM_-awUjTLb8Nc6jtg "素材下载"
