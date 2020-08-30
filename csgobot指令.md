# bot

bot_add 随机增加一个bot

bot_add_ct 增加一名CT

bot_add_t 增加一名T

bot_kick 踢出所有电脑

bot_kill 处死所有电脑

bot_stop 1 bot原地站着不动

bot_freeze 1 冻结所有bot

bot_place 将一个bot放置在此

bot_dont_shoot 1 bot停止射击(但bot被闪会乱开枪)

bot_knives_only bot只能用刀

bot_pistols_only bot只能用手枪

bot_snipers_only bot只能用各种狙

bot_all_weapons bot能用所有武器

bot_difficulty bot难度，数值越高越难

mp_ignore_round_win_conditions 1 删除胜利条件

mp_limitteams 0（关闭双方人数差异值）

mp_autoteambalance 0 （关闭自动平衡，1是打开）

bot_kick，就能踢掉所有bot了

mp_free_armor 无甲/半甲/全甲   |0/1/2

## 无限复活控制台指令

共有4条，注意CT和T是分开设置，前两条命令是允许CT和T在地图中死亡后复活，后两条是设置复活周期时间，比如5秒后将死亡的玩家复活。

```
mp_respawn_on_death_ct 1;
mp_respawn_on_death_t 1;
mp_respawnwavetime_ct 5;
mp_respawnwavetime_t 5;
```

net_graph 1 //显示实时fps ping loss choke var tick

cl_righthand 1/0  //1右手 0左手

cl_updaterate 128

cl_cmdrate 128  //128tick 服务器up cmd指令

exec X  //X为需要读取的cfg名称

 

（绑定键位）

   //绑定ALT键为noclip指令 飞天,穿墙,无敌

bind "F1" "bot_add_ct"    //绑定F1键为bot_add_ct指令 添加一个反恐精英BOT

bind "F2" "bot_add_t"    //绑定F2键为bot_add_t指令 添加一个恐怖分子BOT

bind "0" "bot_place"    //绑定/键为bot_place指令 将一个BOT传送到你面前

 

（机器人跑图）

sv_cheats "1"    //允许使用秘籍

 

ammo_grenade_limit_total "6"    //可携带的手雷数

mp_afterroundmoney "32000"    //每回合结束后增加的金额

mp_autoteambalance "0"    //关闭自动进行人数平衡

mp_buytime "3600"    //购买时间

mp_buy_anywhere "1"    //允许在任何地方购买

mp_death_drop_defuser "0"    //玩家死后不掉落拆弹器

mp_death_drop_grenade "0" //玩家死后不掉落手雷

mp_death_drop_gun "0"    //玩家死后不掉落枪支

mp_forcecamera "0"    //不限制观察者所观看的队伍

mp_free_armor "2"    //免费给予防弹衣和头盔 1=给予防弹衣 2=给予防弹衣和头盔

mp_freezetime "0"    //冻结时间

mp_friendlyfire "1"    //开启右伤

mp_ignore_round_win_conditions "1"    //忽略胜利条件

mp_limitteams "0"    //不限制队伍

mp_maxmoney "32000"    //最大金额

mp_respawn_on_death_ct "1"    //反恐精英死后即刻复活

mp_respawn_on_death_t "1"    //恐怖分子死后即刻复活

mp_roundtime "60.00"    //每回合时间

mp_roundtime_hostage "60.00"    //解救人质地图每回合时间

mp_roundtime_defuse "60.00"    //拆除炸弹地图每回合时间

mp_startmoney "32000"    //初始金额

mp_teammates_are_enemies "1"    //任何人都为目标

mp_warmuptime "0"    //热身时间为0秒

sv_alltalk "1" //开启全局语言

sv_enablebunnyhopping 1 //允许连跳

sv_autobunnyhopping 1 //开启自动连跳

sv_grenade_trajectory "1"    //显示手雷的抛物线

sv_grenade_trajectory_dash "0"    //手雷抛物线的形状

sv_grenade_trajectory_thickness "1"    //手雷抛物线的厚度

sv_grenade_trajectory_time "20"    //手雷抛物线的显示时间

sv_infinite_ammo "2"    //无限备用弹夹

sv_maxvelocity "3500" //设置最大速度为3500

sv_showimpacts "1"    //显示弹道,红色为客户端,蓝色为服务器

sv_showimpacts_time "20"    //弹道的显示时间

 

mp_restartgame "1"    //重新开始游戏

 

bot_kick    //踢出BOT

bot_stop "1"    //BOT为静止状态