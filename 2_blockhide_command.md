# ブロックハイド コマンド
BLOCKHIDE INFO  
攻撃側にめちゃ強い剣を持たせる  
未テスト  

## 共通コマンド
```py
# ゲーム開始時コマンド (tag=team_tagger,team_hider)
impulse > initialize {
    # タグの初期化
    /tag @e remove is_time_up
    # スコアの初期化
    /scoreboard objectives remove cooltime
    /scoreboard objectives add cooltime dummy
    /scoreboard players reset @a game_info
    # 防具立ての初期化
    /kill @e[type=armor_stand,name=gametime]
    /kill @e[type=armor_stand,name=count_hider]
    /kill @e[type=armor_stand,name=count_tagger]
    /summon armor_stand gametime ~~1~
    /summon armor_stand count_hider ~~1~
    /summon armor_stand count_tagger ~~1~
    # 各役職の人数を数える
    /tag @a add not_counted
    /testfor @a[tag=team_hider,tag=not_counted]
    then repeat every 1 tick {
        /scoreboard players add @e[type=armor_stand,name=count_hider] game_info 1
        /tag @r[tag=team_hider,tag=not_counted] remove not_counted
    }
    /testfor @a[tag=team_tagger,tag=not_counted]
    then repeat every 1 tick {
        /scoreboard players add @e[type=armor_stand,name=count_tagger] game_info 1
        /tag @r[tag=team_tagger,tag=not_counted] remove not_counted
    }
    # 各役職の初期化
    >> tagger_initialize
    >> hider_initialize
}

# メインコマンド
lever > main {
    # ゲームが開始できるか確認
    if and {
        /testfor @a[tag=team_tagger]
        /testfor @a[tag=team_hider]
        not /testfor @a[tag=team_tagger,tag=team_hider]
    }
    then /say ゲームを初期化中
    >> initialize

    # [注意] 参加者のインベントリ初期化
    /clear @a[tag=team_tagger]
    /clear @a[tag=team_hider]

    # 隠れ側ゲーム開始 (1分)
    /scoreboard players set @e[type=armor_stand,name=gametime] game_info 60
    >> gametime_counter
    /tp @a[tag=team_hider] <初期位置>

    # 鬼側ゲーム開始 (5分)
    repeat {
        /testfor @e[tag=is_time_up]
    }
    then /scoreboard players set @e[type=armor_stand,name=gametime] game_info 300
    >> gametime_counter
    /tp @a[tag=team_tagger] <初期位置>

    # ゲーム終了判定
    repeat every 1 tick, if or {
        not /testfor @a[tag=team_hider]
        /testfor @a [tag=is_time_up]
    }
    then /title @a title ゲーム終了
    /tp @a[tag=team_tagger] <初期位置>
    /tp @a[tag=team_hider] <初期位置>
    /testfor @a [tag=is_time_up]
    then /title @a subtitle 勝者：隠れ側チーム
    else /title @a subtitle 勝者：鬼側チーム
    >> initialize
}

# クールタイム判定 (score=cooltime,tag=is_ready)
repeat every 1 second {
    # クールタイムが残っていたら、フラグを下げる
    /tag @a[scores={cooltime=1..100}] remove is_ready
    # クールタイムをデクリメントする
    /scoreboard players add @a[scores={cooltime=0..100}] cooltime -1
    # クールタイムが0秒になったら、フラグを立てる
    /tag @a[scores={cooltime=0}] add is_ready
    # ユーザ通知
    /tell @a[scores={cooltime=0}] クールタイム終了
}

# ゲーム時間カウンタ(tag=is_time_up,score=game_info)
repeat every 1 second > gametime_counter {
    /scoreboard players add @e[type=armor_stand,name=gametime] game_info -1
    /tag @e[type=armor_stand,name=gametime,scores={game_info=-100..0}] add is_time_up
}
```

## 隠れ側コマンド
```py
# 初期化(開始時、終了時に呼ぶ)
impulse > hider_initialize {
    # 操作権限を戻す
    /inputpermission set @a lateral_movement enabled
    /inputpermission set @a jump enabled
    # 透明化を解除
    /effect @a clear
    # タグを初期化
    /tag @a remove is_ready
    /tag @a remove is_mimicking
    /tag @a remove is_turning
    /tag @a remove is_releasing
    /tag @a remove is_death
    /tag @a remove is_death_temp
}

# 死亡ハンドラ (tag=is_death,is_death_temp)
repeat every 1 tick {
    # スポーン位置を死んだ場所にする
    execute as @a at @s run spawnpoint @s ~~~
    # 死亡者に死亡タグを付ける
    /tag @a[tag=team_hider] add is_death_temp
    /tag @e[type=player,tag=team_hider] remove is_death_temp
    /tag @a[tag=is_death_temp] add is_death
    # 死亡タグが付いた人の処理(スペク)
    /gamemode spectator @a[tag=is_death]
}

# 判定
repeat every 1 tick {
    # タグの初期化
    /tag @a remove is_sneaking
    /tag @a remove has_space
    /tag @a remove is_landing
    # スニーク判定 (tag=is_sneaking)
    /execute as @a at @s unless entity @s[dx=0,y=~1.5] if entity @s[dx=0,y=~1.4] run tag @s add is_sneaking
    # 十分な空間があるか判定 (tag=has_space)
    /execute as @a at @s if blocks ~~~ ~~2~ 0 -60 0 all run tag @s add has_space
    # 着地判定 (tag=is_landing)
    /execute as @a at @s unless block ~~-1~ air run tag @s add is_landing
}

# ブロック擬態コマンド (tag=is_mimicking,is_turning)
repeat every 1 tick {
    # 必要なタグが付いていたら処理フラグを立てる
    /tag @a[tag=is_sneaking,tag=has_space,tag=is_landing,tag=is_ready,tag=!is_releasing,tag=!is_mimicking,tag=team_hider] add is_turning
    # 操作権限を奪う
    /inputpermission set @a[tag=is_turning] lateral_movement disabled
    /inputpermission set @a[tag=is_turning] jump disabled
    # 透明化
    /effect @a[tag=is_turning] invisibility infinite true
    # 1.2マス上にテレポート
    /execute as @a[tag=is_turning] at @s run tp ~~1.2~
    # 2マス下のブロックを足元にコピー
    /execute as @a[tag=is_turning] at @s run clone ~~-2~ ~~-2~ ~~-1~
    # タグの操作
    /tag @a[tag=is_turning] add is_mimicking # 次の状態フラグを立てる
    /tag @a[tag=is_turning] remove is_turning # 処理フラグを下げる
}

# スニーク解除時の擬態解除コマンド (tag=is_releasing)
repeat every 1 second {
    # 必要なタグが付いていたら処理フラグを立てる
    /tag @a[tag=is_mimicking,tag=!is_sneaking,tag=!is_turning] add is_releasing
    # 前の状態フラグを下げる
    /tag @a[tag=is_releasing] remove is_mimicking
    # 足元のブロックを消す
    /execute as @a[tag=is_releasing] at @s run fill ~~-1~ ~~-1~ air
    # 透明化を解除
    /effect @a[tag=is_releasing] clear
    # 操作権限を戻す
    /inputpermission set @a[tag=is_releasing] lateral_movement enabled
    /inputpermission set @a[tag=is_releasing] jump enabled
    # クールタイムを設定する
    /scoreboard players set @a[tag=is_releasing] cooltime 5 # adjust this value
    # 処理フラグを下げる
    /tag @a remove is_releasing
}
```

## 鬼側コマンド
```py
考え中
```
