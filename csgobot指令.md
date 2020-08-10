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

mp_ignore_win_conditions 1 删除胜利条件

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