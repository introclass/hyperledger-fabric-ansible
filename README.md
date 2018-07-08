## 准备

0. 将要部署到目标环境中的二进制文件复制到output/{{ ENVIROMNT }}/bin/目录中

	mkdir -p output/example.com/
	cd output/example.com/
	wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.1.0/hyperledger-fabric-linux-amd64-1.1.0.tar.gz
	wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.1.0/hyperledger-fabric-linux-amd64-1.1.0.tar.gz.md5
	tar -xvf hyperledger-fabric-linux-amd64-1.1.0.tar.gz

1. 在inventories/{{ ENVIROMNT }}中创建配置文件:

	configtx.yaml
	crypto-config.yaml
	hosts

2. 运行perpare.sh生成证书，以及创世块(可以根据需要修改脚本)：

	./prepare.sh example

注意，`prepare.sh`会使用hyperledger fabric的命令，需要把在本地运行的fabric命令放到`output/bin`目录中。

例如，我是在mac上执行ansible的，下载的是darwin版本的：

	mkdir -p output/bin
	cd output/bin
	wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/darwin-amd64-1.1.0/hyperledger-fabric-darwin-amd64-1.1.0.tar.gz
	wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/darwin-amd64-1.1.0/hyperledger-fabric-darwin-amd64-1.1.0.tar.gz.md5
	tar -xvf hyperledger-fabric-darwin-amd64-1.1.0.tar.gz

## 部署

1. 初始化目标机器

	ansible-playbook -k -i inventories/example.com/hosts -u root deploy_prepare.yml

2. 检测证书设置是否成功

	ansible -i inventories/example.com/hosts -u root  all  -m command -a "pwd"

3. 如果域名没有绑定IP，修改每台机器的/etc/hosts：

	ansible -i inventories/example.com/hosts -u root  all  -m copy -a "src=./inventories/example.com/etc_hosts dest=/etc/hosts"

4. 部署节点

	ansible-playbook -i inventories/example.com/hosts -u root deploy_nodes.yml

5. 部署客户端

	ansible-playbook -i inventories/example.com/hosts -u root deploy_cli.yml

## Fabric初始化

1. 进入member1的管理员目录，对peer0.member1.example.com进行操作：

	cd /opt/app/fabric/cli/user/member1.example.com/Admin-peer0.member1.example.com/
	
	//创建channel，channel只需要创建一次
	./0_create_channel.sh
	
	//加入channel
	./1_join_channel.sh
	
	//设置锚点Peer：
	./2_set_anchor_peer.sh

2. 进入member1的管理员目录，对peer1.member1.example.com进行操作：

	cd /opt/app/fabric/cli/user/member1.example.com/Admin-peer1.member1.example.com
	./1_join_channel.sh

3. 进入member2的管理员目录，对peer0.member1.example.com进行操作：

	cd /opt/app/fabric/cli/user/member2.example.com/Admin-peer0.member2.example.com
	
	//加入channel
	./1_join_channel.sh
	
	//设置锚点Peer：
	./2_set_anchor_peer.sh

## 部署合约

1. 进入member1的管理员目录，对peer0.member1.example.com进行操作：

	cd /opt/app/fabric/cli/user/member1.example.com/Admin-peer0.member1.example.com/
	
	//安装合约
	./3_install_chaincode.sh
	
	//查看已经安装的合约
	./peer.sh chaincode list --installed
	
	//合约实例化，只需要实例化一次
	./4_instantiate_chaincode.sh

2. 在其它Peer上部署合约

	cd /opt/app/fabric/cli/user/member1.example.com/Admin-peer1.member1.example.com/
	./3_install_chaincode.sh

	cd /opt/app/fabric/cli/user/member2.example.com/Admin-peer0.member2.example.com/
	./3_install_chaincode.sh
	
3. 调用合约，写数据

	./6_invoke_chaincode.sh

4. 调用合约，查数据

	./5_query_chaincode.sh

## 管理操作

1. 启动链：

	ansible-playbook -i inventories/example.com/hosts -u root playbooks/manage_start.yml

2. 停止链：

	ansible-playbook -i inventories/example.com/hosts -u root playbooks/manage_stop.yml

3. 清空链上所有数据：

	ansible-playbook -i inventories/example.com/hosts -u root playbooks/manage_rebuild.yml

4. 销毁链：

	ansible-playbook -i inventories/example.com/hosts -u root playbooks/manage_destroy.yml
