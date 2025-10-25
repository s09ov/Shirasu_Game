# 宝探し コマンド
宝探し全部を実装するのは骨が折れる（宝箱再生成・タグ付与等を宝箱ごとに設置する必要があるため）    
下は集計コマンドのみの実装。  
最初に/clear @a[tag=treasure_hunt]でインベントリを削除しておく必要がある。  

## 集計コマンド
```py
# tag = treasure_huntがついていること前提
impulse {
  # スコアを初期化
  /scoreboard players reset @a[tag=treasure_hunt] treasure_hunt_score
  # 集計コマンド
  /scoreboard players add @a[tag=treasure_hunt,hasitem={item=tripwire_hook}] treasure_hunt_score 100
  /scoreboard players add @a[tag=treasure_hunt,hasitem={item=iron_block}] treasure_hunt_score 100
  ...
  # インベントリを消去
  /clear @a[tag=treasure_hunt]
  # 集計結果表示
  /scoreboard objectives setdisplay sidebar treasure_hunt_score descending
}
```
