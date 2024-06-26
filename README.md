# isucon-my-sheet

ISUCON の練習で使ったコマンドや参考になった記事のまとめ

## 環境構築

### ssh しようとしたときに使った秘密鍵で、BadPermisshion のエラーが出た時

秘密鍵のパーミッションを 400 にする
```
$ chmod 400 ~/.ssh/秘密鍵名.pem
```

### isucon ユーザーでローカルから SSH する

isucon ユーザーの~/.ssh/authorized_keys に Github に登録している公開鍵を追加
https://github.com/iwashi623.keys

```
$ mkdir .ssh && touch ~/.ssh/authorized_keys
```

### ssh-agent とローカルの秘密鍵を利用して、isucon サーバーから GitHub の認証を通す

```
以下、ローカル
--------------------------------------
**秘密鍵をssh-agentに登録**
$ ssh-add -k ~/.ssh/使いたい秘密鍵
**秘密鍵が登録されたかの確認**
$ ssh-add -l
**ssh-agentを使ってインスタンスへssh**
ssh -A  -i ~/.ssh/使いたい秘密鍵 isucon@<インスタンスIP>

以下、インスタンス内
--------------------------------------
成功すると以下のように表示される
$ ssh git@github.com
PTY allocation request failed on channel 0
Hi iwashi623! You've successfully authenticated, but GitHub does not provide shell access.
Connection to github.com closed.
```

### GitHub に公開鍵登録(新しい秘密を作りたい時以外は不要)

- https://qiita.com/shizuma/items/2b2f873a0034839e47ce

```
 $ git config --global user.name "iwashi623"
 $ git config --global user.email GitHubメールアドレス
```

### Git コマンドのエイリアスを作成

```
$ vi ~/.gitconfig

---------以下を貼り付け---------
[user]
    name = iwashi623
    email = <githubメールアドレス>

[alias]
    s = status
    ci = commit
    l = log --oneline
    co = checkout
    br = branch
```

## サーバーの状態把握
- CPU

```
 $ lscpu
 または、
 $ cat /proc/cpuinfo
```

- メモリ
  メガバイト表示、ギガバイト表示は`-g`

```
 $ free -m
```

- サービス

```
 $ systemctl list-units --type=service
```

### シンボリックリンク
```
貼る　　　　　　　参照先ディレクトリ                        置く場所　　　
ln -s /home/isucon/private_isu/webapp/etc /etc/mysql/isu-mysql

剥がす.  置き場所
unlink /etc/mysql/isu-mysql
```

### scp
```
ローカルのファイルを転送
$ scp webapp/mysql/db/2_DummyChairData.sql isucon@<IPアドレス>:/home/isucon/isuumo/webapp/mysql/db/

リモートのディレクトリを転送
$ scp -r isucon@<IPアドレス>:/home/isucon/isuumo/webapp/mysql/db webapp/mysql/ 
```

### 環境把握

- htop インストール
```
$ sudo apt install htop
```
`shift + m`でメモリ使用順、`shift + P`で CPU 使用順でソートできる

- 開いているPortの確認
```
$ sudo netstat -anp
```

- file 名で検索
```
例）sudo find /  -name "default.conf"
```

- systemd の設定ファイルを探す
```
$ sudo find / -type f -name "*isucari*"

直接コマンドで設定ファイルも見れる
$ sudo systemctl cat isucari.go.service
```

- systemctl で status 確認
```
sudo systemctl status nginx.service
```

### Systemd に環境変数を渡す

```
$ vi /etc/systemd/system/isucari.golang.service
[Unit]
Description=My Go Application

[Service]
ExecStart=/path/to/your/go-binary
Environment="VAR1=value1" "VAR2=value2" <- ここを編集する
Restart=always
User=username
Group=groupname

[Install]
WantedBy=multi-user.target



$ sudo systemctl daemon-reload && sudo systemctl start isucari.golang && sudo systemctl enable isucari.golang
```

### systemdの設定ファイルを書き換えたら
```
設定ファイルの読み込み
$ systemctl daemon-reload
```

### アプリを restart したい時

```
sudo systemctl disable --now isucholar.go.service
sudo systemctl enable --now isucholar.go.service
sudo systemctl status isucholar.go.service
```

## nginx
### 設定確認
```
# エラーログなどを確認できる
sudo nginx -t
```

### alp

- https://github.com/tkuchiki/alp
- [alp のインストールと nginx のログ設定](https://nishinatoshiharu.com/install-alp-to-nginx/)
- [色々まとまっている](https://kazegahukeba.hatenablog.com/entry/2019/09/13/015113)

```
$ wget https://github.com/tkuchiki/alp/releases/download/v1.0.21/alp_linux_amd64.zip
$ unzip alp_linux_amd64.zip
$ sudo install ./alp /usr/local/bin
```

ログのローテーション

```
$ echo -n "" > /var/log/nginx/access.log && sudo chmod 777 /var/log/nginx/access.log
過去ログ削除と再起動
$ sudo rm /var/log/nginx/access.log && sudo systemctl reload nginx
```

実行

```
$ alp -f /var/log/nginx/access.log
パターンで絞り込み(ここでは/posts/{post_id}と/@{hogehoge}を絞り込み)
$ alp -f /var/log/nginx/access.log --aggregates='posts/[0-9]+,/@\w+'' --sum -r
$ alp --sum -r -f /var/log/nginx/access.log --aggregates='/api/estate/[0-9]+,/api/chair/[0-9]+,/api/recommended_estate/[0-9]+' > hoge.txt
```

参考

- https://github.com/Nagarei/isucon11-qualify-test/issues/1#issuecomment-912392530

### レスポンスをキャッシュする
[参考](https://github.com/oystersjp/yubikega-isucon10-qualify/commit/2868646c550bd3e8680e38aa5aa6579986dccf03)
```
proxy_cache_path /var/cache/nginx keys_zone=zone1:1m max_size=100m inactive=5m;

server {
    root /home/isucon/isucon10-qualify/webapp/public;
    listen 80 default_server;
    listen [::]:80 default_server;

    location ~ ^/api/estate/[0-9]+$ {
            proxy_cache       zone1;
            proxy_cache_key   $scheme$proxy_host$uri$is_args$args;
            proxy_cache_valid 200 30s;
            proxy_pass http://localhost:1323;
    }

    location /api/estate/search {
            proxy_cache       zone1;
            proxy_cache_key   $scheme$proxy_host$uri$is_args$args;
            proxy_cache_valid 200 30s;
            proxy_pass http://localhost:1323;
    }

    location /api {
            proxy_pass http://localhost:1323;
    }

    location /initialize {
            proxy_pass http://localhost:1323;
    }

    location / {
            root /www/data;
    }
}
```

### 問答無用で500エラーを返す
```
if ($http_user_agent ~* ^isubot ) {
  return 503;
}
```

## MySQL

### スキーマの確認
```
$ sudo mysqldump --compact --no-data isuconp | grep -v "^SET" | grep -v "^/\*\!" | perl -ple 's@CREATE TABLE @\nCREATE TABLE @g'
```

### pt-query-digest

- MySQL のスローログ設定
  [参考](https://nishinatoshiharu.com/mysql-slow-query-log/)

```
sudo vi /etc/mysql/my.cnf
~~~
[mysqld]
  slow_query_log = 1
 slow_query_log_file = /var/log/mysql/slow.log
 long_query_time = 0
 disable-log-bin
 innodb_buffer_pool_size = 1GB # ディスクイメージをメモリ上にバッファさせる値をきめる設定値
 innodb_flush_log_at_trx_commit = 2 # 1に設定するとトランザクション単位でログを出力するが 2 を指定すると1秒間に1回ログファイルに出力するようになる
 innodb_flush_method = O_DIRECT # データファイル、ログファイルの読み書き方式を指定する(実験する価値はある)
~~~

sudo service mysql restart
sudo mysql
mysql> show variables like 'slow%';
+---------------------+-------------------------+
| Variable_name       | Value                   |
+---------------------+-------------------------+
| slow_launch_time    | 2                       |
| slow_query_log      | ON                      |
| slow_query_log_file | /var/log/mysql/slow.log |
+---------------------+-------------------------+
3 rows in set (0.01 sec)

```

- ログローテート

```
$ now=`date +%Y%m%d-%H%M%S` && sudo mv /var/log/mysql/slow.log /var/log/mysql/slow.log.$now && sudo mysqladmin flush-logs
```



- [pt-query-digest](https://nishinatoshiharu.com/percona-slowquerylog/)

```
$ wget https://www.percona.com/downloads/percona-toolkit/3.0.10/binary/debian/xenial/x86_64/percona-toolkit_3.0.10-1.xenial_amd64.deb
$ sudo apt-get install libdbd-mysql-perl libdbi-perl libio-socket-ssl-perl libnet-ssleay-perl libterm-readkey-perl
$ sudo dpkg -i percona-toolkit_3.0.10-1.xenial_amd64.deb
OR

$ cd /usr/local/bin
$ sudo curl -LO percona.com/get/pt-query-digest
$ sudo chmod +x pt-query-digest

$ sudo pt-query-digest /var/log/mysql/slow.log
```

### INDEX を貼る

```
$ mysql
## tableの詳細確認
show create table table_name;

## commentsテーブルのINDEX確認
show index from comments;

## INDEX追加
ALTER TABLE テーブル名 ADD INDEX インデックス名(カラム名);

## 複合INDEX追加
ALTER TABLE posts ADD INDEX index_posts_on_created_at_updated_at(created_at, updated_at);

## INDEX削除
ALTER TABLE テーブル名 DROP INDEX インデックス名;
```

### DB分割
#### 移行先DBホスト
1. ユーザーと権限の追加
```sql
-- ユーザーの作成
CREATE USER `isucon`@`172.31.%` IDENTIFIED BY 'isucon';
-- 権限の追加
GRANT ALL PRIVILEGES ON `isuumo`.* TO `isucon`@`172.31.%`;

-- DBのユーザー一覧取得
SELECT user, host FROM mysql.user;
-- DBのユーザー権限確認('isucon'@'172.31.%'はユーザー名)
SHOW GRANTS FOR 'isucon'@'172.31.%';
```
2. bind-addressを0.0.0.0にする
```
$ sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf

# Instead of skip-networking the default is now to listen only on
# localhost which is more compatible and is not less secure.
bind-address            = 0.0.0.0 <- こうする
```

#### アプリケーションホスト
```bash
// 環境変数設定を確認
$ sudo systemctl cat isuumo.go
```
DBのHostの設定をDBのHostにしたいサーバーのPrivateIPアドレスに切り替える｡アプリで使っている箇所があればそちらも切り替える

## pprof
### 導入
1. [こんな感じにアプリに入れる](https://github.com/hoge-times/isucon9-qualify-20240217/pull/5)
2. ベンチ実行後､Initializeが終わったタイミングで以下のコマンド事項
   ```
    $ go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
   ```
3. ローカルでファイルを展開する
   ```
    $ go tool pprof -http 127.0.0.1:9090 ./pprof.isucari.samples.cpu.002.pb.gz
   ```

## log

```
$ journalctl -xe | grep  isucari
# systemd
$ sudo tail -n 1000 /var/log/syslog | grep hogehoge
```

## (練習用)Goのバージョンを変える
```
goのパスを確認しておく
$ which go
$ printenv | grep GOROOT

goのパッケージを取ってくる
$ wget https://go.dev/dl/go1.21.5.linux-amd64.tar.gz

既存のGo削除
$ rm -rf /home/isucon/local/go

取ってきたGoのパッケージを解凍してGoコマンドが読みに行くPathに展開する
$ tar -C /home/isucon/local/ -xzf go1.21.5.linux-amd64.tar.gz
```
