#plantUML
#DB 

## 1
0か1のみ関連する
![[Pasted image 20260104114613.png]]

必ず1つ関連する
![[Pasted image 20260104114620.png]]


# 多
0以上関連する
![[Pasted image 20260104114624.png]]

0以上関連する
![[Pasted image 20260104114627.png]]


# 覚え方
- 0も含まれるときは丸が付く
- 0が含まれないときは横線が付く
- 1のときは横線
- 他の時は鳥の足

例:
	必ず一つ関連
	→横線 + 横線
	0~多の関連
	→丸 + 鳥足


# plantUML記法

```plantUML
@startuml
--o| // 0か1のみ関連する
--|| // 必ず1つ関連する 
--o{ // 0以上関連する(多) 
--|{ // 1以上関連する(多)
@enduml
```

```plantUML
@startuml

entity "ユーザー" as User {
  + user_id : int　//+属性が主キーとなる
  --
  * name : string
  * email : string
}

entity "プロフィール" as Profile {
  + profile_id : int
  --
  * user_id : int
  * address : string

}

' 1対1の関係
User ||--|| Profile
@enduml
```
![D:\docker\out\DB_Practice\design\ER図\ER図.png](file:///d%3A/docker/out/DB_Practice/design/ER%E5%9B%B3/ER%E5%9B%B3.png)

