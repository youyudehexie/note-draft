#Pomelo
##客户端API
###pomelo.init(opt, cb)  
连接pomelo服务器

+ opt 连接参数 {host:host, port: port, log: true}
+ cb回调函数

###pomelo.request(handler, msg, cb)

     pomelo.request('connector.entryHandler.enter'{username: username}, cb); 

发送请求给服务端，第一次向connector发送请求时，要将session信息注册上去

+ handler 服务器的鸭子类型
+ msg 发送给handler函数所带的参数
+ cb回调函数

###pomelo.on(event, cb)

    //update user list
  pomelo.on('onAdd', function(data) {
		var user = data.user;
		tip('online', user);
		addUser(user);
	});

监听Pomelo发送的事件

+ event 时间名字
+ cb 回调函数
