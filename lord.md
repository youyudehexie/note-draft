#Lord游戏流程分析

js/ui/main.js

    function main(){
        clientManager.init();  //初始化客户端，注册与登录前的备用信息等等
        setDefaultUser();  //设置默认登录用户，方便测试
        heroSelectView.init(); //英雄选择框
        addEvents();  //监听登录事件,点击登录后触发 clientManager.entry 函数
    }

   function addEvents(){
	    document.getElementById('login').addEventListener('click', clientManager.entry, false);   //用户进入
	    document.getElementById('heroSelectBtn').addEventListener('click', clientManager.register, false);  //用户注册处理
	}

js/ui/clientManage

    /**
	     * enter game server
	     * route: connector.entryHandler.entry
	     * response：
	     * {
	     *   code: [Number],
	     *   player: [Object]
	     * }
     */
    function entry(host, port, token, callback) {
      // init socketClient
      // TODO for development
      if(host === '127.0.0.1') {
        host = config.GATE_HOST;
      }
      pomelo.init({host: host, port: port, log: true}, function() {
        pomelo.request('connector.entryHandler.entry', {token: token}, function(data) {
          var player = data.player;

          if (callback) {
            callback(data.code);
          }

          if (data.code == 1001) {
            alert('Login fail!');
            return;
          } else if (data.code == 1003) {
            alert('Username not exists!');
            return;
          }

          if (data.code != 200) {
            alert('Login Fail!');
            return;
          }

          // init handler
          loginMsgHandler.init();
          gameMsgHandler.init();

          if (!player || player.id <= 0) {
            switchManager.selectView("heroSelectPanel");
          } else {
            afterLogin(data);
          }
        });
      });
    }

pomelo.init连接pomelo服务器，pomelo.request 请求pomelo服务器进行处理connector.entryHandler.entry

game-server/app/servers/connector/entryHandler.js

	/**
	 * New client entry game server. Check token and bind user info into session.
	 *
	 * @param  {Object}   msg     request message
	 * @param  {Object}   session current session object
	 * @param  {Function} next    next stemp callback
	 * @return {Void}
	 */
	pro.entry = function(msg, session, next) {
	var token = msg.token, self = this;

	if(!token) {
		next(new Error('invalid entry request: empty token'), {code: Code.FAIL});
		return;
	}

	var uid, players, player;
	async.waterfall([
		function(cb) {
			// auth token
			self.app.rpc.auth.authRemote.auth(session, token, cb);
		}, function(code, user, cb) {
			// query player info by user id
			if(code !== Code.OK) {
				next(null, {code: code});
				return;
			}

			if(!user) {
				next(null, {code: Code.ENTRY.FA_USER_NOT_EXIST});
				return;
			}

			uid = user.id;
			userDao.getPlayersByUid(user.id, cb);
		}, function(res, cb) {
			// generate session and register chat status
			players = res;
			self.app.get('sessionService').kick(uid, cb);
		}, function(cb) {
			session.bind(uid, cb);
		}, function(cb) {
			if(!players || players.length === 0) {
				next(null, {code: Code.OK});
				return;
			}

			player = players[0];
			session.set('areaId', player.areaId);
			session.set('playername', player.name);
			session.set('playerId', player.id);
			session.on('closed', onUserLeave.bind(null, self.app));
			session.pushAll(cb);
		}, function(cb) {
			self.app.rpc.chat.chatRemote.add(session, player.userId, player.name,
				channelUtil.getGlobalChannelName(), cb);
		}
	], function(err) {
		if(err) {
			next(err, {code: Code.FAIL});
			return;
		}

		next(null, {code: Code.OK, player: players ? players[0] : null});
	});
};

1. rpc远程调用认证服务器
2. 获取用户ID
3. 从session中踢走其他客户端登录该账户的用户
4. 绑定新的服务器session
5. 利用session存放地图ID,玩家ID，玩家名字等信息到session下
6. 绑定close事件，当用户关闭的时候，执行onUserLeave
7. 将用户添加到后端的频道上
8. 返回给客户端

客户端收到服务器登录成功信息后，执行

	// init handler
	loginMsgHandler.init(); //登录逻辑的事件绑定
	gameMsgHandler.init();  //游戏逻辑的时间绑定

js/handler/loginMsgHandler.js

loginMsgHandler Event

+ onKick 踢走用户事件
+ disconnect 断开连接事件
+ onUserLeave 用户离开事件

js/handler/gameMsgHandler.js

gameMsgHandler
 
+ onChangeArea  改变的Area服务器，去到另外一张地图
+ onAddEntities 服务器通知客户端添加或删除实物
+ onDropItems 响应跌下来的装备
+ onRemoveEntities 移除实物
+ onMove 移动行为
+ onPathCheckout 更新移动路线
+ onUpgrade 人物升级
+ onUpdateTaskData 背包更新，收到消息后，更新背包
+ onTaskCompleted 任务完成
+ onRemoveItem 移除物品
+ onPickItem 捡起物品
+ onNPCTalk 与NPC聊天
+ onCheckoutTask 看任务菜单
+ onAttack 攻击
+ onRevive 复活

目前，我对lord的游戏逻辑更感兴趣，先从gameMsgHandler来展开分析

###onChangeArea

	/**
	 * Handle change area message
	 * @param  data {Object} The message
	 */
	pomelo.on('onChangeArea', function(data) {
		if(!data.success) {
			return;
		}
		clientManager.loadResource({jsonLoad: false}, function() {
			pomelo.areaId = data.target;
			pomelo.request("area.playerHandler.enterScene",{uid:pomelo.uid, playerId: pomelo.playerId, areaId: pomelo.areaId}, function(msg) {
				app.init(msg);
			});
		});
	});

js/handler/npcHandler.js
	
	/**
	 * Change area action.
	 */
	function changeArea(params) {
	var areaId = pomelo.areaId, target = params.target;
	pomelo.request("area.playerHandler.changeArea", {
			uid:pomelo.uid,
			playerId: pomelo.playerId, 
			areaId: areaId,
			target: target
		}, function(data) {
	  pomelo.emit('onChangeArea', data);
		});
	};

这个函数与onChangeArea的事件相关

pomelo服务端

app/servers/handler/playHander.js

	//Change area
	handler.changeArea = function(msg, session, next) {
		var areaId = msg.areaId;
		var target = msg.target;
	
		var req = {
	    areaId: areaId,
	    target: target,
	    uid: session.uid,
	    playerId: session.get('playerId'),
	    frontendId: session.frontendId
	  };
	
		world.changeArea(req, session, function(err) {
			next(null, {areaId: areaId, target: target, success: true});
		});
	};
 
app/domain/world.js

	/**
	 * Change area, will transfer a player from one area to another
	 * @param args {Object} The args for transfer area, the content is {playerId, areaId, target, frontendId}
	 * @param cb {funciton} Call back funciton
	 * @api public
	 */
	exp.changeArea = function(args, session, cb) {
		var uid = args.uid;
		var playerId = args.playerId;
		var areaId = args.areaId;
		var target = args.target;
		var player = area.getPlayer(playerId);
		var frontendId = args.frontendId;
		area.removePlayer(playerId);
		//messageService.pushMessage({route:'onUserLeave', code: 200, playerId: playerId});
	
		var pos = this.getBornPoint(target);
	
		player.areaId = target;
		player.x = pos.x;
		player.y = pos.y;
		userDao.updatePlayer(player, function(err, success) {
			if(err || !success) {
				err = err || 'update player failed!';
				utils.invokeCallback(cb, err);
			} else {
				session.set('areaId', target);
				session.push('areaId', function(err) {
					if(err){
						logger.error('Change area for session service failed! error is : %j', err.stack);
					}
					utils.invokeCallback(cb, null);
				});
			}
		});
	};


1. 将用户移除地图
2. 设置新的出生点
3. 从用户数据库中更新用户的地点资料
4. 完成地图改变后，执行area.playerHandler.enterScene 服务器端代码，该代码主要功能在跳转到该地图后，进行地图初始化，加入到该地图的聊天频道。


###onAddEntities

	/**
	 * Handle player update event
	 * @param {Object} params Params for player update, the content is : {watchers, id}
	 * @return void
	 * @api private
	 */
	function onPlayerUpdate(params) {
		var player = area.getEntity(params.id);
		if(player.type !== EntityType.PLAYER) {
			return;
		}
	
		var uid = {sid : player.serverId, uid : player.userId};
	
		if(params.removeObjs.length > 0) {
	    messageService.pushMessageToPlayer(uid, 'onRemoveEntities', {'entities' : params.removeObjs});
		}
	
		if(params.addObjs.length > 0) {
			var entities = area.getEntities(params.addObjs);
			if(entities.length > 0) {
	      messageService.pushMessageToPlayer(uid, 'onAddEntities', entities);
			}
		}
	}

1. 当AOI发现附近玩家附近有新物体的时候，服务器告诉客户端添加该物体。

###onDropItems
	/**
	 * Player get rewards after task is completed.
	 * the rewards contain equipments and exprience, according to table of figure
	 *
	 * @param {Player} player
	 * @param {Array} ids
	 * @api public
	 */
	taskReward.reward = function(player, ids) {
		if (ids.length < 1) {
			return;
		}
	
		var i, l;
		var tasks = player.curTasks;
		var pos = player.getState();
		var totalItems = [], totalExp = 0;
	
		for (i = 0, l=ids.length; i < l; i++) {
			var id = ids[i];
			var task = tasks[id];
			var items = task.item.split(';');
			var exp = task.exp;
			for (var j = 0; j < items.length; j++) {
				totalItems.push(items[j]);
			}
			totalExp += exp;
		}
	
		var equipments = this._rewardItem(totalItems, pos);
		this._rewardExp(player, totalExp);
	
		for (i = 0, l=equipments.length; i < l; i ++) {
			area.addEntity(equipments[i]);
		}
	
		messageService.pushMessageToPlayer({uid:player.userId, sid : player.serverId}, 'onDropItems', equipments);
	};


##addEntity

	/**
	 * Add entity to area
	 * @param {Object} e Entity to add to the area.
	 */
	exp.addEntity = function(e) {
		if(!e || !e.entityId) {
			return false;
		}
	
		if(!!players[e.id]) {
			logger.error('add player twice! player : %j', e);
			return false;
		}
	
		entities[e.entityId] = e;
		eventManager.addEvent(e);
	
		if(e.type === EntityType.PLAYER) {
			getChannel().add(e.userId, e.serverId);
			aiManager.addCharacters([e]);
	
			aoi.addWatcher({id: e.entityId, type: e.type}, {x : e.x, y: e.y}, e.range);
			players[e.id] = e.entityId;
			users[e.userId] = e.id;
		}else if(e.type === EntityType.MOB) {
			aiManager.addCharacters([e]);
	
			aoi.addWatcher({id: e.entityId, type: e.type}, {x : e.x, y: e.y}, e.range);
		}else if(e.type === EntityType.NPC) {
	
		}else if(e.type === EntityType.ITEM) {
			items[e.entityId] = e.entityId;
		}else if(e.type === EntityType.EQUIPMENT) {
			items[e.entityId] = e.entityId;
		}
	
		aoi.addObject({id:e.entityId, type:e.type}, {x: e.x, y: e.y});
		return true;
	};


##addEvent


##channelService


## aoi.addWatcher()




##FAQ

1，这里的entities是自己组的，但因为新的lord里面有了protobuf，这个会根据其内容自动压缩。应该是你自己加的属性不符合protobuf中消息定义的格式而导致压缩/解压失败。你可以修改serverProtos.json里面的onAddEntities的定义使之符合你定义的格式，如果不用protobuf压缩的话直接删掉也可以，这样就会采用非压缩的格式传输。

2，因为物体的位置变化会导致他所在AOI区域（tower）的变化以及他观察区域的变化，这样就需要通知AOI模块更新其位置。你这里是特殊情况，只有一个AOI区域，所以就不需要了。不过要在AOI区域切换时应该会用到。这两个方法在lord中最终会产生removeEntities和AddEntities事件，告诉客户端自动的删除视野外的实体，并添加视野内的实体。

3，pushMessageByAOI是当在某一个位置发生非全局消息时，需要通过AOI来通知视野内的所有观察者。在你这种基于房间的地图实际上就是通知这个房间内的所有玩家就可以了，可以不用。

根据你的设计，建议你每个房间建立一个channel，通过这些channel来广播，不需要使用AOI模块。因为AOI模块适用于通知者和被通知者会根据位置和视野动态变化的环境，无法或者很难构建出固定的channel。而在你的情况下，因为每个room都相对不变，使用channel会更方便一些。



