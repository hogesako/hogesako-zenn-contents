---
title: "ISUCON12にくすサポISUCON部として参加し、予選突破しました"
emoji: "🦑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [isucon]
published: true
---
# サマリ
ISUCON12にくすサポISUCON部として参加し、スコア25666、19位で予選突破しました。

# チーム構成
- tondol: アプリを実装する人
- すっちー: インフラ関連をやる人
- hogesako(自分): SQLをチューニングする人。たまにアプリに口を出す。

# 事前準備〜前日
2週間くらい前にISUCON本の問題を解いたり、ISUCON11予選の問題を解き直したり、サウナに行ったり、寿司を食ったりしていました。
tondolがMakefileでalpやpt-query-digestの環境構築やログローテ等をできるようにしてくれていました。

前日はよく眠れるように、全員に対して禁酒を発令。
tondolはいかさこハウスに前日入りしていたので、駅で売っていたヤクルト1000や、グリシン3000を飲ませるなどしていました。

# 当日やったこと
## MySQL移行
以下のような理由で10:40くらいから着手していました。
- 事前に準備してきたpt-query-digestを用いたサイクルを回せないのが何より問題だと感じたこと
- alpで判明したいちばん重いボトルネックはtondolが対応していること
- tondolのチューニングは1号機で実施、自分のMySQLへの移行検証は3号機で実施と言う感じで、違うマシンを使えば影響なく進められそうだったこと
- 失敗してもDROP TABLEするだけっぽかったこと

### 脳内検証
MySQLへの移行には以下2点の課題をクリアしないといけないと考えていました。
1. アプリとしてMySQLに移行しても動くか
SQLという言語と、sqlxやdriverなどで抽象化されているし、よっぽどなDBMS固有の使い方してなければいけるっしょと思っていたのでOKとしました

2. initializeの30秒制限に引っかからないか
    - competitionとplayerは大きくないのでDROPしたあと作り直せばよさそう。
    - player_scoreはINSERTしかしていなさそうだし、visit_historyと同じように
    ```sql
    DELETE FROM player_score WHERE tenant_id > 100;
    DELETE FROM player_score WHERE created_at >= '1654041600';
    ```
    すればいいのでは！？と思いOKにしました。**(注：この判断は間違っています)**

### 移行開始
とりあえず用意されていたスクリプトを使い、雑なシェルスクリプトを作って全テナント分を出力し、
```bash:export.sh
for ((i=1 ; i<=100 ; i++))
do
./sqlite3-to-sql ../../initial_data/$i.db > tenant_init/$i.sql
done
```

**手作業**でMySQLにCREATE TABLEしたあと、雑なシェルスクリプトで入れてみた
```bash:import.sh
ISUCON_DB_HOST=${ISUCON_DB_HOST:-127.0.0.1}
ISUCON_DB_PORT=${ISUCON_DB_PORT:-3306}
ISUCON_DB_USER=${ISUCON_DB_USER:-isucon}
ISUCON_DB_PASSWORD=${ISUCON_DB_PASSWORD:-isucon}
ISUCON_DB_NAME=${ISUCON_DB_NAME:-isuports}

for ((i=1 ; i<=100 ; i++))
do
mysql -u"$ISUCON_DB_USER" \
		-p"$ISUCON_DB_PASSWORD" \
		--host "$ISUCON_DB_HOST" \
		--port "$ISUCON_DB_PORT" \
		"$ISUCON_DB_NAME" < tenant_init/$i.sql
done
```
shellを実行している間はtondolの作業を見守ったりコードを読んだりしていました。
たしか30分くらいで入りきった気がします。

データを入れたあとはとりあえずお試しで、adminDBと同じ雰囲気でdocker-composeの環境変数から接続情報を渡し、
```diff yaml
  webapp:
    build: ./go
    environment:
      ISUCON_DB_HOST: 127.0.0.1
      ISUCON_DB_PORT: 3306
      ISUCON_DB_USER: isucon
      ISUCON_DB_PASSWORD: isucon
      ISUCON_DB_NAME: isuports
+     TENANT_DB_HOST: 192.168.0.13
+     TENANT_DB_PORT: 3306
+     TENANT_DB_USER: isucon
+     TENANT_DB_PASSWORD: isucon
+     TENANT_DB_NAME: isuports
```

DB接続コードをadminDB部分から**がっつりコピペ**
```diff go
// テナントDBに接続する
func connectToTenantDB(id int64) (*sqlx.DB, error) {
-	p := tenantDBPath(id)
-	db, err := sqlx.Open(sqliteDriverName, fmt.Sprintf("file:%s?mode=rw", p))
-	if err != nil {
-		return nil, fmt.Errorf("failed to open tenant DB: %w", err)
-	}
-	return db, nil
+	config := mysql.NewConfig()
+	config.Net = "tcp"
+	config.Addr = getEnv("TENANT_DB_HOST", "127.0.0.1") + ":" + getEnv("TENANT_DB_PORT", "3306")
+	config.User = getEnv("TENANT_DB_USER", "isucon")
+	config.Passwd = getEnv("TENANT_DB_PASSWORD", "isucon")
+	config.DBName = getEnv("TENANT_DB_NAME", "isuports")
+	config.ParseTime = true
+	dsn := config.FormatDSN()
+	return sqlx.Open("mysql", dsn)
}
```
その他SQLite固有のコードをちょこっと削除するなどしていたらベンチマークが通ったので一安心という感じでした。

### 文字化け
ベンチは通ったものの、ブラウザで見るとマルチバイト文字が文字化けしているという問題が発生。CREATE TABLE時の文字コード指定と、import時の文字コードを指定してもう一度移行し直さなければなりませんでした・・・。
```diff bash:import.sh
ISUCON_DB_HOST=${ISUCON_DB_HOST:-127.0.0.1}
ISUCON_DB_PORT=${ISUCON_DB_PORT:-3306}
ISUCON_DB_USER=${ISUCON_DB_USER:-isucon}
ISUCON_DB_PASSWORD=${ISUCON_DB_PASSWORD:-isucon}
ISUCON_DB_NAME=${ISUCON_DB_NAME:-isuports}

for ((i=1 ; i<=100 ; i++))
do
mysql -u"$ISUCON_DB_USER" \
		-p"$ISUCON_DB_PASSWORD" \
		--host "$ISUCON_DB_HOST" \
		--port "$ISUCON_DB_PORT" \
+               --default-character-set=utf8mb4 \
		"$ISUCON_DB_NAME" < tenant_init/$i.sql
done
```
12:40くらいには文字化けを直した状態で、3テーブルすべてMySQLに移行完了していたと思います。

### initializeが遅ぇ
initializeが遅くてたまにfailするという問題も発生。
player、competitionは `./sqlite3-to-sql` で出力したsqlファイルだと1行ずつINSERTしていたので、MySQLに入れたtableをさらにmysqldumpで出力しなおしたsqlファイルをinitializeに使用するように変更しました。
また、visit_history、player_scoreをcreated_at条件で消している部分がスロークエリログに出ていたので、インデックスを貼ったりしていました。initializeのためだけにインデックスを貼って更新系が遅くなるのはいかがなものかと思いつつ、競技終了後の再試でinitializeできなくて失格とかよりはましか〜とか考えていたきがします。
```sql
CREATE INDEX vh_created_at on visit_history(created_at);
CREATE INDEX ps_created_at ON player_score(created_at);
```

### player_score、DELETEもしとるやん・・・
脳内検証の時点ではplayer_scoreはINSERTしかしていなさそうだし、
```sql
DELETE FROM player_score WHERE tenant_id > 100;
DELETE FROM player_score WHERE created_at >= '1654041600';
```
すればいいのでは！？という判断をしていました。

ベンチが通っていたので深く考えていなかったのですが、競技終了後に他の方のエントリや解説を読むと普通にDELETEしてからINSERTしていますね！？
1. (おそらく)DELETEしたあと過去のデータも含めてCSVに入ってきている
2. csvにはcreated_atも入っている
3. ベンチマーク開始時にcreated_atの条件で消している
ために(運良く)動いていたと思われます。
ちなみにDELETE INSERTをトランザクション化までは時間切れでたどり着けていません。

## そのほかやったこと
MySQL移行完了後は粛々と以下のようなことをしていました。
- pt-query-digestにPREPAREが出ていたので `config.InterpolateParams = true` にする
- pt-query-digestにでてきたスロークエリを潰す
- tondolとペアプロ

# 終わってみての所感
- コードを読めてない部分が多々ありすぎる。改善しないと本戦で戦える気がしませんね

# おまけ
くすサポISUCON部のメンバーで参加するのは5回目で、やっとこさ本戦に出られたか！という感じです。
初回参加はISUCON8予選と楠田亜衣奈さんの「アイナンダ！」リリースイベントがモロかぶりしており、新幹線の中や名古屋近鉄パッセの屋上（リリースイベント会場）で参加したのを覚えています(そのときは絶賛リリイベ真っ最中というチーム名でした)。
その後毎年参加する中で、惜しいところまでいけたり、まったく歯がたたず惨敗したりもしました。
やはり予選突破or惜しいところまでいけるのは「推測するな、計測せよ」をきちんと守れたときでした。
**「推測するな、計測せよ」をしたとしても予選突破できるとは限らないけれど、「推測するな、計測せよ」をしないと惨敗するのはマジで本当。**
