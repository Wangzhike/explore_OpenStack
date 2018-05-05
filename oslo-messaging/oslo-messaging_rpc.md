# oslo-messaging的RPC使用

## 1. 几个基本概念：	
1. Transport	

2. Target	
	> A Target encapsulates all the information to identify where a message should be send or what message a server is listening for.	
	> The default target used for all subsequent calls and casts is supplied to
    the RPCClient constructor.  The client uses the target to control how the
    RPC request is delivered to a server.  If only the target's topic (and
    optionally exchange) are set, then the RPC can be serviced by any server
    that is listening to that topic (and exchange).  If multiple servers are
    listening on that topic/exchange, then one server is picked using a
    best-effort round-robin algorithm.  Alternatively, the client can set the
    Target's ``server`` attribute to the name of a specific server to send the
    RPC request to one particular server.  In the case of RPC cast, the RPC
    request can be broadcast to all servers listening to the Target's
    topic/exchange by setting the Target's ``fanout`` property to ``True``. 

	具有如下属性：	
	1. exchange		
		> A scope for topics. Leave unspecified to default to the `control_exchange` configuration option.	

		对应于RabbitMQ中的exchange，负责消息的转发，其`exchange_type=topic`(通过`sudo rabbitmqctl list_exchanges`命令，看到类型为`topic`)，此处的`tpoic`不是下文的`topic`，而是RabbitMQ中exchagne的`exchange_type`属性。		
	
	2. topic	
		> A name which identifies the set of interfaces exposed by a server. Multiple servers may listen on a topic and messages will be dispatched to one of the servers selected in a best-effort round-robin fashion (unless fanout is `True`).	

		对应于RabbitMQ的消息的routing_key的一部分，另外一部分是下文的`server`选项。如果设置了`server`选项，那么就构成了唯一的routing_key，所以可以唯一指定一个RPCServer；否则，使用`best-effort round-robin algorithm`，选出监听该topic的服务器中的某一台作为目的地。	
		> If only the target's topic (and optionally exchange) are set, then the RPC can be serviced by any server that is listening to that topic (and exchange).  If multiple servers are listening on that topic/exchange, then one server is picked using a best-effort round-robin algorithm.  Alternatively, the client can set the Target's ``server`` attribute to the name of a specific server to send the RPC request to one particular server.	

	3. namespace	

		Identifies a particular RPC interface (i.e. set of methods) exposed by a server.	

	4. version

		RPC interface have a major.minor version number associated with them.	
	
	5. server

		RPC Clients can request that a message be directed to a specific server, rather than just one of a pool of servers listening on the topic.	

	6. fanout

		布尔值。	
		> The call() method is used to invoke RPC methods that return a value. Since only a single return value is permitted it is not possible to call() to a fanout target.	
		> In the case of RPC cast, the RPC request can be broadcast to all servers listening to the Target's topic/exchange by setting the Target's ``fanout`` property to ``True``.	
	
	在不同的应用场景下，构造Target对象需要不同的参数：

	| 应用	|	必选参数	| 可选参数	|
	| :---:	| :---:			| :---:		|
	| RPC Server	| topic, server		| exchange	|
	| RPC endpoint	| None				| namespace, version	|
	| PRC client	| topic				| all other attributes	|
	| Notification Server	| topic		| exchange, all other attributes ignored	|
	| Notifier		| topic				| exchange, all other attributes ignored	|


3. Server

	A RPC server exposes a number of endpoints, each of which contains a set of methods which may be invoked remotely by clients over a given transport.	

4. RPC Client

	通过RPC Client，可以远程调用RPC Server上的方法。远程调用时，需要提供一个字典对象来指明调用的上下文，调用方法的名字和传递给调用方法的参数(用字典表示)。有`cast`和`call`两种远程调用方式。通过cast方式远程调用时，请求发送后就直接返回了；通过call方式远程调用，需要等响应从服务器返回。	
	> The default target used for all subsequent calls and casts is supplied to the RPCClient constructor.  The client uses the target to control how the RPC request is delivered to a server.  If only the target's topic (and optionally exchange) are set, then the RPC can be serviced by any server that is listening to that topic (and exchange).  If multiple servers are listening on that topic/exchange, then one server is picked using a best-effort round-robin algorithm.  Alternatively, the client can set the Target's ``server`` attribute to the name of a specific server to send the RPC request to one particular server.  In the case of RPC cast, the RPC request can be broadcast to all servers listening to the Target's topic/exchange by setting the Target's ``fanout`` property to ``True``.	

5. Notifer

	Notifer用来通知某种transport发送通知消息。	

6. Notification Listener

	Notification Listener和Server类似，一个Notification Listener对象可以暴露多个endpoint，每个endpoint包含一组方法。但是与Server对象中的endpoint不同的是，这里的endpoint中的方法对应通知消息的不同优先级。	

## 2. RPC Client与RPC Server之间的消息模型

![RPC_routing](https://github.com/Wangzhike/explore_OpenStack/raw/master/oslo-messaging/pictures/RPC_routing.png)	

openstack exchange的`routing_key`为topic.[server]，其中`server`为可选部分。正如上面所述，如果server为空，只指定topic，那么采用`best-effort round-robin`算法从监听该topic的所有服务器中选择一台为客户端服务；如果也指定了server选项，那么将选择监听该topic且名字为server选项的服务器来为客户端服务。		

如上图所示，在没有指定server选项的情况下，假设最终选定RPC Server1来服务。那么RPC Server1将执行远程调用，取得调用执行结果，然后创建一个`direct`类型的exchange，并和RPC Client创建的`Reply_#`的消息队列进行绑定，其`binding_key`即为该exchange的`routing_key`，将调用执行结果打包为消息，发送给exchange，再由exchange将消息转发到`Reply_#`的消息队列，由RPC Client取得该消息，并根据其中的`msg_id`信息识别出该消息属于哪次的RPC调用。该过程的消息模型如下：		
![RPC_reply](https://github.com/Wangzhike/explore_OpenStack/raw/master/oslo-messaging/pictures/RPC_reply.png)	

## 3. RPC Client和RPC Server程序说明消息模型	

client代码：	

```python
from oslo_config import cfg
import oslo_messaging as messaging
import sys

servername = sys.argv[1] if len(sys.argv) >= 2 else None
print("servername: %s" % servername)
transport = messaging.get_transport(cfg.CONF)
# call调用对应的Target不能是fanout
target = messaging.Target(topic='test', server=servername)
client = messaging.RPCClient(transport, target)
client.call(ctxt={}, method='test', arg='hello')
print("Call test success")
cctxt = client.prepare(namespace='control', version='2.0', fanout=True)
cctxt.cast({}, 'stop')
print("Cast stop success")
```

server代码：	

```python
from oslo_config import cfg
import oslo_messaging
import time
import sys

class ServerControlEndpoint(object):

    target = oslo_messaging.Target(namespace='control', version='2.0')

    def __init__(self, server):
        self.server = server

    def stop(self, ctx):
        print('stop')
        if self.server:
            self.server.stop()

class TestEndpoint(object):

    def test(self, ctx, arg):
        print(arg)
        return arg

servername = sys.argv[1] if len(sys.argv) >= 2 else 'server1'
print("server: %s" % servername)
transport = oslo_messaging.get_transport(cfg.CONF)
target = oslo_messaging.Target(topic='test', server=servername)
endpoinds = [
    ServerControlEndpoint(None),
    TestEndpoint(),
]
server = oslo_messaging.get_rpc_server(transport, target, endpoinds, executor='blocking')

try:
    server.start()
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    server.stop()
    print("Stopping server")

server.wait()
```

1. 打开一个终端，运行`python server.py server1`		

	终端输出(忽略警告信息)：	
	```shell
    server: server1	
	```

	执行`sudo rabbitmqctl list_queues`结果：	
	```shell
    Listing queues ...
	test	0
	test.server1	0
	test_fanout_44a2de88ee674b4d8a83361e3b38e011	0
	```

	执行`sudo rabbitmqctl list_exchanges`结果：		
	```shell
    Listing exchanges ...
			direct
	amq.direct	direct
	amq.fanout	fanout
	amq.headers	headers
	amq.match	headers
	amq.rabbitmq.log	topic
	amq.rabbitmq.trace	topic
	amq.topic	topic
	openstack	topic
	test_fanout	fanout	
	```

	可见，在创建一个RPC Server时，就会创建一个名为`openstack`的类型为`topic`的exchange，以及创建一个名字为`topic_fanout`的类型为`fanout`的exchange，即`test_fanout`。	

2. 打开另一个终端，运行`python server.py server2`	

	终端输出(忽略警告信息)：	
	```shell
    server: server2 
	```

	执行`sudo rabbitmqctl list_queues`结果：	
	```shell
	Listing queues ...
	test	0
	test.server1	0
	test.server2	0
	test_fanout_1345b2fc32b64c69b90a20a50b7039a1	0
	test_fanout_44a2de88ee674b4d8a83361e3b38e011	0
	```

	执行`sudo rabbitmqctl list_exchanges`结果：		
	```shell
	Listing exchanges ...
			direct
	amq.direct	direct
	amq.fanout	fanout
	amq.headers	headers
	amq.match	headers
	amq.rabbitmq.log	topic
	amq.rabbitmq.trace	topic
	amq.topic	topic
	openstack	topic
	test_fanout	fanout
	```

3. 打开另一个终端，执行`python client.py`	

	终端输出：	
	```shell
    servername: None
	Call test success
	Cast stop success	
	```

	server1终端输出：	
	```shell
	hello
    stop
	```

	server2终端输出：	
	```shell
	stop
	```

	执行`sudo rabbitmqctl list_queues`结果：	
	```shell
	Listing queues ...
	reply_5a16a4a86b664c058fba9b929b6d8c17	0
	test	0
	test.server1	0
	test.server2	0
	test_fanout_1345b2fc32b64c69b90a20a50b7039a1	0
	test_fanout_44a2de88ee674b4d8a83361e3b38e011	0	
	```

	执行`sudo rabbitmqctl list_exchanges`结果：		
	```shell
	Listing exchanges ...
			direct
	amq.direct	direct
	amq.fanout	fanout
	amq.headers	headers
	amq.match	headers
	amq.rabbitmq.log	topic
	amq.rabbitmq.trace	topic
	amq.topic	topic
	openstack	topic
	reply_5a16a4a86b664c058fba9b929b6d8c17	direct
	test_fanout	fanout
	```

4. 在client终端，继续执行`python client.py server2`		

	终端输出：	
	```shell
	servername: server2
	Call test success 
	Cast stop success
	```

	server1终端输出：	
	```shell
	stop
	```

	server2终端输出：	
	```shell
	hello
	stop
	```

	执行`sudo rabbitmqctl list_queues`结果：	
	```shell
	Listing queues ...
	reply_5a16a4a86b664c058fba9b929b6d8c17	0
	reply_cdf32d06f44743b6a9852b003b710bc3	0
	test	0
	test.server1	0
	test.server2	0
	test_fanout_1345b2fc32b64c69b90a20a50b7039a1	0
	test_fanout_44a2de88ee674b4d8a83361e3b38e011	0
	```

	执行`sudo rabbitmqctl list_exchanges`结果：		
	```shell
	Listing exchanges ...
			direct
	amq.direct	direct
	amq.fanout	fanout
	amq.headers	headers
	amq.match	headers
	amq.rabbitmq.log	topic
	amq.rabbitmq.trace	topic
	amq.topic	topic
	openstack	topic
	reply_5a16a4a86b664c058fba9b929b6d8c17	direct
	reply_cdf32d06f44743b6a9852b003b710bc3	direct
	test_fanout	fanout
	```

5. 在client终端，继续执行`python client.py server1`		

	终端输出：	
	```shell
	servername: server1
	Call test success
	Cast stop success
	```

	server1终端输出：	
	```shell
	hello
	stop
	```

	server2终端输出：	
	```shell
	stop
	```

	执行`sudo rabbitmqctl list_queues`输出：	
	```shell
	Listing queues ...
	reply_207029192bf44eb3b92168b795c632b7	0
	reply_5a16a4a86b664c058fba9b929b6d8c17	0
	reply_cdf32d06f44743b6a9852b003b710bc3	0
	test	0
	test.server1	0
	test.server2	0
	test_fanout_1345b2fc32b64c69b90a20a50b7039a1	0
	test_fanout_44a2de88ee674b4d8a83361e3b38e011	0
	```

	执行`sudo rabbitmqctl list_exchanges`输出：		
	```shell
	Listing exchanges ...
			direct
	amq.direct	direct
	amq.fanout	fanout
	amq.headers	headers
	amq.match	headers
	amq.rabbitmq.log	topic
	amq.rabbitmq.trace	topic
	amq.topic	topic
	openstack	topic
	reply_207029192bf44eb3b92168b795c632b7	direct
	reply_5a16a4a86b664c058fba9b929b6d8c17	direct
	reply_cdf32d06f44743b6a9852b003b710bc3	direct
	test_fanout	fanout
	```

执行`cast`调用前，使用`prepare`方法修改RPC Client的Target对象的属性，可以修改的属性包括：exchange、topic、namespcae、version、server、fanout、timeout、version_cap和retry。修改后的target属性只在prepare()方法返回的对象中有效。	
由于prepare()方法设置`fanout=True`，如上所述，RPC Client的cast调用会广播给所有监听该topic的服务器，从而引发它们每一个上的远程过程调用。		


