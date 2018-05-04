# oslo.messaging使用	

## 1. RPC	
几个基本概念：	
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

		对应于RabbitMQ中的exchange，负责消息的转发，其`exchange_type=direct`(猜测，因为现象最符合)，此处的`tpoic`不是下文的`topic`，而是RabbitMQ中exchagne的`exchange_type`属性。		
	
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


