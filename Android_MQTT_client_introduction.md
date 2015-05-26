#MQTT Client简介

##MQTT简介

##主要接口与类

MQTT Client以两种方式向MQTT Server通信，分别为同步模式和异步模式。两者的接口分别为IMqttClient和IMqttAsyncClient，同步模式为对异步模式的很薄的一层封装。

###MqttSecurityException

当安全相关的配置有异常或者client没有权限进行某种操作时，抛出该类异常。

###MqttPersistenceException

如果自己实现了相应的持久化接口，但是该实现的调用出现问题时，抛出该类异常。

###MqttException

与服务器通信出现错误时，抛出该类异常。

###MqttPingSender

该接口用于向MQTT broker（注：应该是server？）发送心跳包。它提供接口有

1. void 	init(ClientComms comms) 用于初始化，传入一个connection对象

1. void 	schedule(long delayInMilliseconds) 下次发送心跳包的时间

1. void 	start() 开始执行心跳发送

1. void 	stop() 结束执行心跳发送，在MQTT Client出现错误或者连接被中断时，该函数被调用。

###IMqttAsyncClient

这个接口可以用来与MQTT Server异步的通信。它实现的接口主要有

1. connect

1. publish

1. subscribe

1. unsubcribe

1. disconnect

每一种方法有两种类型，第一种形如

	IMqttToken token = asyncClient.method(parms)
         
示例代码如下

	IMqttToken conToken;
	conToken = asyncClient.client.connect(conToken);
	do some work...
	conToken.waitForCompletion();
         
我们也可以用同步的方式编写这段代码。

	IMqttToken token;
	token = asyncClient.method(parms).waitForCompletion();
         
第二种类型更接近JavaScript的premise方式，它提供callback机制来应对传输成功或者失败的情况。

	IMqttToken token method(parms, Object userContext, IMqttActionListener callback)
         
示例代码如下

	IMqttToken conToken;
	    conToken = asyncClient.connect("some context",new new MqttAsyncActionListener() {
			public void onSuccess(IMqttToken asyncActionToken) {
				log("Connected");
			}
			public void onFailure(IMqttToken asyncActionToken, Throwable exception) {
				log ("connect failed" +exception);
			}
		});
          

这里没有使用额外的userContext对象，在实际使用中，callback函数里面需要用到userContext。

###IMqttClient

该端口用于与MQTT服务器端同步通信。它实现的接口主要有

1. connect

1. publish

1. subscribe

1. unsubcribe

1. disconnect

该类接口没有那种带有callback的调用方式

###IMqttActionListener

在异步通信完成后，MQTT Client根据执行的成功或者失败调用对应的函数，它拥有的接口包括

1. void 	onFailure(IMqttToken asyncActionToken, Throwable exception)

1. void 	onSuccess(IMqttToken asyncActionToken)

需要注意的事，如果连接的时候cleanSession字段被设置为false且Qos被设置为1或2时，当前网络不可用时会调用onFailure，但是再有网络的时候还是会尝试将消息重发。

###IMqttToken

提供机制用于追踪异步通信的完成情况，一个通信被认为是正在进行中，除非isComplete返回值为真或者getException返回值为非空。该接口类主要的方法有

1. getActionCallback

1. IMqttAsyncClient 	getClient()

1. MqttException 	getException()

1. int[] 	getGrantedQos()

1. int 	getMessageId() 返回与这个token相关联的message的ID

1. String[] 	getTopics() 返回该token关联的请求正在操作的topic

1. Object 	getUserContext() 返回该请求相关联的context

1. boolean 	isComplete()

1. void 	setActionCallback(IMqttActionListener listener)

1. void 	setUserContext(Object userContext)

1. void 	waitForCompletion

###IMqttDeliveryToken

该接口继承自IMqttToken，即使connection或者client被重启它仍然有效，它的主要接口有

1. MqttMessage 	getMessage() 返回与它相关联的token.

它的使用方式有两种，分别对应同步或者异步两种方式。

1. IMqttAsyncClient.getPendingDeliveryTokens()

1. MqttCallback.deliveryComplete(IMqttDeliveryToken)

使用时需要注意

1. 连接的cleanSession字段需要设置为false

1. 一个通信被认为是正在进行中，除非isComplete返回值为真或者getException返回值为非空。

###MqttCallback

该接口定义了特定几种类型事件的callback，主要的接口包括。

1. void 	connectionLost(Throwable cause) 在服务器失去联系时被调用。

1. void 	deliveryComplete(IMqttDeliveryToken token) 在信息发送完成并且受到回执的时候被调用。

1. void 	messageArrived(String topic, MqttMessage message) 在收到服务器发来的消息时被调用。

对于同步请求，调用方式为
	
	IMqttClient.setCallback(MqttCallback)

对于异步请求，调用方式为

	IMqttAsyncClient.setCallback(MqttCallback)

###MqttConnectOptions

提供客户端向服务器建立连接的参数，主要的方法有

1. void 	setCleanSession(boolean cleanSession) 设置客户端和服务器在重新连接或者重新启动后，是否纪录上一次的状态。

1. void 	setConnectionTimeout(int connectionTimeout) 设置多长时间后断开连接，默认是30秒，在设置为0时客户端会等待无限长的时间。

1. void 	setKeepAliveInterval(int keepAliveInterval) 设置"keep alive"时间间隔，该字段默认是60秒，如果设置为0，客户端keepalive机制失效。MQTT Client用这个机制来判断服务器是否还在，而不需要等到TCP/IP连接超时，客户端需要保证在一个"keep alive"至少有一次通信发生，如果此期间没有通信，客户端需要发一个ping给服务器。

1. void 	setMqttVersion(int MqttVersion)

1. void 	setPassword(char[] password) 设置此次通信的密码。

1. void 	setServerURIs(String[] array) 设置MQTT Client可以连接的服务器列表。

1. void 	setSSLProperties(Properties props) 设置加密通信的相关信息。

1. void 	setUserName(String userName) 设置建立该次连接使用的用户名。

1. void 	setWill(MqttTopic topic, byte[] payload, int qos, boolean retained) 设置遗嘱，在连接意外中断时，服务器会向自己发送一条消息。

1. protected static int 	validateURI(String srvURI) 验证URI.

###MqttMessage

它的实例用于保存消息传递的option和payload(要传递的内容，使用byte[]表示)

它的主要方法有：
           
1. void 	clearPayload()

1. int 	getQos()

1. boolean 	isDuplicate() 返回这条信息是否可能是重复的，只有服务器发来的信息可能设置这一字段。

1. boolean 	isRetained() //注：该字段的文档没看懂

1. protected  void 	setDuplicate(boolean dup)
           
1. protected  void 	setMutable(boolean mutable) 该消息可否被修改

1. void 	setPayload(byte[] payload)

###MqttTopic

用于代表一个主题，用来发布或者订阅，主要的接口包括：

1. getName()
          
1. MqttDeliveryToken 	publish(byte[] payload, int qos, boolean retained)

###MqttPersistable和MqttClientPersistence

用于客户端的持久化。

