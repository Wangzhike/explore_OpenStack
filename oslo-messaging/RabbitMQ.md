# RabbitMQ    

## 1. 安装    

### 1.1 安装Erlang/OTP    
由于RabbitMQ是由Erlang编写的，所以需要安装Erlang/OTP才能运行。由于Debian和Ubuntu自带的Erlang版本太老，所以需要手动添加Erlang的官方仓库。    

1. 添加Erlang Version Pinning    
  在`/etc/apt/preferences.d/`目录下，新建文件`erlang`，内容如下：    
  ```shell
  # /etc/apt/preferences.d/erlang
  Package: erlang*
  Pin: version 1:20.1-1
  Pin-Priority: 1000

  Package: esl-erlang
  Pin: version 1:20.1.7
  Pin-Priority: 1000 
  ```

2. 添加Erlang官方仓库    
  在`/etc/apt/sources.list`文件中新增：    
  ```shell
  deb https://packages.erlang-solutions.com/debian xenial contrib
  ```

  为`apt-secure`添加Erlang官方仓库的公钥：    
  ```shell
  wget https://packages.erlang-solutions.com/debian/erlang_solutions.asc
  sudo apt-key add erlang_solutions.asc
  ```

3. 安装Erlang    
  刷新仓库缓冲：    
  ```shell
  sudo apt-get update
  ```

  安装erlang或esl-erlang：    
  ```shell
  sudo apt-get install erlang 或 sudo apt-get install esl-erlang
  ```

### 1.2 添加RabbitMQ的Bintray仓库    
和Erlang一样，Debian和Ubuntu自带的`rabbitmq-server`版本太低，所以需要手动添加Erlang的Bintray仓库或Package Cloud仓库。以下使用Bintray仓库。    

1. 添加bintray的apt源    
  ```shell
  echo "deb https://dl.bintray.com/rabbitmq/debian xenial main" | sudo tee /etc/apt/sources.list.d/bintray.rabbitmq.list
  ```

2. 添加公钥：    
  ```shell
  wget -O- https://dl.bintray.com/rabbitmq/Keys/rabbitmq-release-signing-key.asc |
       sudo apt-key add -
  ```

## 1.3 安装rabbitmq-server    
1. 刷新包列表    
  ```shell
  sudo apt-get update
  ```

2. 安装    
  ```shell
  sudo apt-get install rabbitmq-server
  ```

## 1.4 运行RabbitMQ Server    
执行：    
```shell
sudo service rabbitmq-server start
```

## 2. RabbitMQ基本使用    
RabbitMQ分为RabbitMQ Server和支持多种语言的RabbitMQ client。下面以Pika Python Client为例，说明其使用。    

1. Producer    
  生产者负责发送消息    

2. Exchange		

	一端从生产者接收消息，另一端负责将消息发送到消息队列。当exchange接收到消息后，根据`exchange_type`的不同，决定将消息发送到某一个队列，或是多个队列，或者是丢弃该消息。	
	exchange的类型有：direct、topic、headers和fanout。		

3. Queue    
  消息队列就是RabbitMQ内部的邮箱，虽然消息经过应用和RabbitMQ，但是其只能被存储在消息队列中。消息队列就是一个巨大的消息池，其大小受内存和磁盘大小的限制。许多生产者可以发送消息到一个队列，而多个消费者尝试从一个队列中接收消息。		

4. Consumer    
  消费者等待从队列中接收消息    

查看RabbitMQ的消息队列和每个队列中的消息个数：    
```shell
sudo rabbitmqctl list_queues
```

查看RabbitMQ的exchange的个数、名字及类型：	
```shell
sudo rabbitmqctl list_exchanges
```

## 2.1 Hello World    
单个生产者发送一条消息，单个消费者接收该消息然后将其打印到屏幕上。	

消息模型如下：	
![python-one-overall](https://github.com/Wangzhike/explore_OpenStack/raw/master/oslo-messaging/pictures/RabbitMQ/python-one-overall.png)

1. Sending	

	发送消息的基本步骤：	
	1. 建立与RabbitMQ server的通信	

	2. 发送消息前确认接收消息的队列存在		

		通过`channel.queue_declare(queue='hello')`创建一个名为`hello`的消息队列。如果发送消息到一个不存在的消息队列，RabbitMQ会丢弃该消息。		
	
	3. 使用空字符串标识的默认exchange	

		RabbitMQ消息模型的核心观念是生产者从不直接发送消息到消息队列，而是发送给exchange。默认的exchange由空字符串标识，并将发送给它的消息传递到和它的`routing_key`选项同名的消息队列中。	

	4. 关闭连接		

		关闭连接，会刷新网络缓冲区并确保消息被发送到RabbitMQ。	


