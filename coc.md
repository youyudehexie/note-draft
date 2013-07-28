##资源升级模块
资源模块分为三种情况讨论，资源升级前，资源升级中，资源升级后

###一、资源升级前

####用户流程
点击升级按钮->检查工人和资源是否符合条件->扣除资源->开始升级
客户端
1.  点击升级的时候，通过查询升级条件的配置文件，判断用户的资源是否满足条件且工人数量也符合条件。
2.	判断用户符合条件后，扣除用户的资源，并将用户对资源的操作行为，写入动作数组。创建用户资源升级倒计时，倒计时长可查询配置文件。
3.	倒计时结束后，客户端画面显示为升级成功后的资源。
4.	每5秒扫描一次动作数组，判断数组内，是否存在用户行为，当发现用户行为时，将该用户行为post到服务器。

注意：为了避免客户端频繁向服务器发出请求，所以对用户的操作行为，5秒集中发一次。

####服务器端
1.	接收客户端发送的用户行为。
2.	获取当前的服务器时间，记录为资源升级的起始时间，由于客户端是延时发送用户行为，可在该时间减去部分时间，减少误差。
3.	查询配置文件，读出该资源升级所需要的时间，然后计算出完成升级的时间，最后修改实时内存中的资源的状态为升级中，扣除相应的资源。
4.	将资源的信息添加到redis下的有序集合，完成时间是有序集合排序关键值，而资源的归属标记为redis的关键字，当我们向有序集合获取一个成员的时候，可以通过关键字获取该资源的附加信息。
5.	服务器创建一个定时监听器，每秒都向redis的有序集合查询，与服务器当前的时间作比较，若资源达到目标时间，则提取信息，删除该键值，根据该对象的关键字，获取升级资源的附加信息，根据附加信息，修改数据库的资源信息，若该用户的缓存还存在，则同时修改缓存信息和数据库信息。

###二、资源升级中

####用户流程

进入画面->资源显示升级未完成->看到倒计时在动。

####客户端

1.	用户进入画面，从服务器获取该用户家园信息，服务器时间和资源状态等。
2.	判断某一个资源状态，在升级中，从该附加信息中，获取完成升级的时间，计算该完成时间和服务器时间的差值，利用差值，创建一个升级中的计时器。

####服务器

1.	收到用户家园请求
2.	检查内存是否存在用户信息，若存在，读取后发送到用户，若不在，从数据库读取，写入内存后，发送到用户。

###三、完成资源升级

####用户流程

进入画面->资源显示升级完毕

####客户端

1.	用户进入画面，从服务器获取该用户家园信息，服务器时间和资源状态等。
2.	读取资源信息，发现并没有资源在建设中，正常显示建筑

####服务器

1.	收到用户家园请求
2.	检查内存是否存在用户信息，若存在，读取后发送到用户，若不在，从数据库读取，写入内存后，发送到用户。

注意：当用户离线时，资源升级任务已经存放到redis的有序集合中处理，当到资源到达了完成时间后，服务器就会改写对应的升级后的信息。

##资源采集模块

###用户流程

点击资源采集->画面显示用户获得多少资源->资源版上，显示新添加的资源

###客户端

1.	初始化地图的时候，获取服务器的时间和上一次更新资源的时间。
2.	用户点击时计算可获取的资源多少，可获取资源公式如下：
3.	
              可获取资源最大值  	  (增加资源> =可获取资源最大值)
可获取资源

              生产时间 * 资源增长速度 (增加资源 < 资源最大值)

生产时间 = 客户端当前时间 – 上一次更新资源时间

4．资源增长速度可以从配置文件中获取，根据计算公式，计算出用户获取的资源，并将资源添加到用户的资源版面上并保证不超过资源版面上的最大值，同时记录用户的采集行为到用户的动作数组。
5. 客户端记录这次更新的客户端时间，方便下次使用，避免向服务器请求。
6. 每5秒扫描一次动作数组，判断数组内，是否存在用户行为，当发现用户行为时，将该用户行为post到服务器。

注意：为了避免客户端频繁向服务器发出请求，所以对用户的操作行为，5秒集中发一次。另外，客户端本地时间并不可靠，不一定与服务器同步，所以初始化的时候传入服务器时间，如果客户端有能力，应该以那个时间作为基准时间。

###服务器

1.	接收客户端发送的用户行为。
2.	读取服务器的当前时间，由于客户端是延时发送用户行为，可在该时间减去部分时间，减少误差。
3.	根据上述可获取资源公式，计算出可获取的资源。将资源添加到用户系统资源上，并判断资源没有大于用户可容纳资源的范围。
4.	将计算结果写入对应实时缓存键值中。

注意：服务器计算可获取资源的时候，并没有采用客户端发送过来的时间，因为客户端的时间不可靠，可能会导致作弊行为。

##排行榜模块

###用户流程

点击排行榜->显示前500位排名者->点击榜单中的用户可以访问中户的家园
点击排行榜->显示玩家个人排名

###客户端

1.	向服务器获取全服前500位选手
2.	获取名单和其相关信息后，在客户端进行缓存，下次用户访问榜单的时候，不再向服务器发出请求，直接读取内存，并显示。
3.	当用户点击榜单中的玩家时，向服务器发送该玩家的ID，服务器响应返回改玩家的家园信息。
4.	根据玩家的家园信息，对玩家的家园地图进行初始化。

###服务器

1.	接受客户端请求，从排行服务器中，获取前500位的玩家信息，并发送到客户端。
2.	从有序集合的内存里，读取改玩家ID的排名，并发送到客户端。
3.	当客户端发送查询其他用户信息ID的请求时，根据ID查询缓存中，如果有该用户信息，直接读取，如果没有，则从服务器中提取。
排行榜信息维护
排行榜实际上，采用redis的有序集合，用户的分数是该有序集合排行的权重值，用户的ID作为键值。
1.	当有新玩家进入时，添加信息到该有序集合，保证该集合拥有全服的名单。
2.	当玩家分数变动的时候，修改该集合里面玩家的对应分数，有序集合能根据分数产生排名。
3.	为了性能考虑，可以每5分钟或者更长时间，缓存一次有序集合生成的排名500人的列表，减轻计算量。

##战斗匹配模块
匹配规则
+ 离线玩家
+ 玩家没有受到系统保护
+ 经过算法后综合得分接近

###用户流程
点击战斗->匹配出对手

###客户端
1.	发送用户ID，请求到服务器战斗匹配
2.	获取对手信息

###服务器
1.接收客户端请求
2.查询用户ID的实力分数
3.根据实力分数，在一定分数范围里匹配相应键值redis有序集合。
4.在匹配分数范围里的用户，提取到数组里，从数组中随机出一个索引值，根据索引值，获取该用户的信息，根据该用户信息读取，读取对应的战争保护字段，因为该字段是定时失效字段，时长为用户受到保护的时间，如果字段不存在，则说明用户没受到保护，存在则受到保护，踢除该用户后，再数组里面，随机出一个合适的用户，知道匹配到用户为止，为了避免进入死循环，设置匹配的次数不能超过数组的长度，超过，则返回无法匹配。
5.将匹配到的用户作为敌人信息发送到客户端。

###匹配模块维护
1.Connector服务器会一直向客户端发送心跳包，当检测到用户离线后，读取该离线用户的战斗实力分数，将其写入到redis的有序集合，将用户的战斗分数作为有序集合排序的权重，键值为该用户对应的ID。当检测到用户登录时，从集合中删除该用户的键值，保证了集合中的用户都是离线用户。
2.过大的集合会导致有序集合计算变慢，因此，服务器根据多个分数段分散到不同的集合键值，假设我们分为三组，大于100分的键值为rank:0，小于100大于50的rank:1，小于50的则是rank:2，根据玩家的分数值，到对应的集合去找合适的用户去寻找合适的对手，因为检测到在线的时候，会删除对应集合的键值，所以是动态分配的，高可用。
3.每次战斗的时候或者通过道具，产生战斗保护时间段，根据时间长度，设置一个会过期的键值，当键值过期的时候，说明保护已经失效了。

##战争模块

###用户流程
进入战斗画面->在规定范围和时间内投放兵力->士兵根据各自的特性进行战斗->战斗结束后，结算战斗得失

###客户端
战斗场景是一个巨大的循环，场景的函数会在每秒循环执行函数一次，直到战斗结束，战斗结束的条件有四个，1.防守方建筑被消灭 2.攻击方撤退 3攻击方的士兵全部被消灭 4.战斗超时。.

###防守方
1.根据战斗对手的家园信息，初始化地图。
2.每一次循环，防守塔对象都检测附近地图数组，从近到远，有没敌方目标，当发现敌人后，根据防御塔的属性，攻击敌人，当目标脱离攻击范围或者被消灭后，转移到下一个攻击目标。
3.对于资源类建筑，被攻击时，会产生资源掠夺效果，根据该建筑占据的资源总量，通过资源量/建筑血量 的公式计算出每一下攻击所掠夺的资源并显示在客户端上，并计算到掠夺资源数量级记分牌上。
4.当防守方被攻击时，会根据 被攻击损耗的血量/所有建筑物的总血量，计算出破坏率，根据破坏率，计算出用户所获得的奖杯。

###进攻方
1.创建战斗倒计时。
2.生成可以投放兵力的范围，投兵的范围约地方建筑的4-5格范围外，将地方建筑与地方建筑范围外格子标记为禁止投兵。
3.确认投兵时，没有超出规定的战斗准备时间范围内。
4.当客户端检测到用户合法投放士兵后，记录投兵的地点，投兵的时间，投放的兵种。
5.基于A*的寻路算法，根据不同特性兵种的特性，设置不同的不行路线，步兵的目标地点是基地，老鼠的目标地点粮仓或金库，胖子的目标地点是防御塔，士兵遇到阻碍物时，根据不同的兵种特性，判断阻碍物是否能被攻击，是否需要攻击。当目标被消灭后，将设置其他优先级不高的目标，行走路线也是通过A*算法获得。
6.当士兵的生命值为0的时候，被消灭。

##战斗结束
将资源记变动数据，进攻方投兵时间，投兵地点，投兵兵种和保护不被进攻等信息，发送到服务器。

###服务器
1.	接受客户端请求
2.	资源变动信息，如黄金，食物，损失的士兵将写入双方的实时内存中
3.	投兵地点，投兵兵种，投兵时间，记录到响应的内存数据键值里，主要为录像系统做准备
4.	如果防守方失败，保存其不被进攻的状态值，利用redis的过期键值方式，设置时长为不被进攻时间时长
5.	写入用户的战斗历史

##战斗录像回放模块
在类COC游戏中，并没有随机攻击力，闪避，等不确定因素，只要确定好投放兵力的地点，只要确定兵种，时间和对手，那么同一套计算系统，计算出来的结果，都是一样的，所以回放的时候，服务器只需要发送兵种和投放兵种的坐标和时间，就能通过算法模拟出相同的战斗场面。

###用户流程
打开战斗历史->播放战斗历史

###客户端
1.向服务器获取战斗历史
2.根据战斗记录的ID，向服务器获取战斗记录信息
3.从服务器获取，战斗的投兵地点，投兵兵种，投兵时间，沿用相同的战斗算法，在客户端跑一次，实现录像效果。

###服务器端
1.从数据库中匹配出该用户最近的战斗历史，并将结果缓存，避免反复查询。发送到客户端。
2.根据ID，从数据库读取战斗记录里的战斗的投兵地点，投兵兵种，投兵时间发送到客户端。