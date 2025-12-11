## 手順書
1. 課題概要Raspberry Pi OS 上に Docker Compose を用いて Web サーバー (Apache) と データベース (MySQL) の2コンテナ構成のアプリケーション環境を構築し、ホストの 8080番ポート から Web サイトにアクセスできること
発展要素例B: Docker Compose による Web+DBの2コンテナ
構成完成条件http://<Raspberry Pi の IP アドレス>:8080 にアクセスすると、Webサイトの初期ページが表示される
docker compose up コマンドで環境全体が起動・停止できる
データベースデータがホストのボリュームに永続化されている 
2. 前提条件項目詳細対象 OS とバージョンRaspberry Pi OS
実行環境Raspberry Pi 
利用ツール・パッケージ管理手段apt, docker, docker compose 
ネットワーク条件固定ローカル IP アドレス、ドメイン無し、HTTPS 不可 
前提ユーザーpi ユーザーで作業し、sudo 権限が付与されていること 
## 手順
1. パッケージの更新 
sudo apt update 
sudo apt upgrade -y
2. Docker と Docker Compose のインストールDocker のインストール (資料 14.2.2参照) 
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo なしで docker を実行できるようにする (資料 14.2.3 参照) 
## 現在のユーザを docker グループに追加
sudo usermod -aG docker $USER
## または newgrp docker を実行
newgrp docker
3. 基本的なファイアウォール設定 (UFW)
SSH ポート (22番)と Web 公開ポート (8080番) のみ許可する (資料 11.3 に基づき)
1. デフォルト設定: 入ってくる通信はすべて拒否
sudo ufw default deny incoming
## 2. SSH (22 番ポート) を許可
sudo ufw allow ssh 
## 3. Web 公開ポート (8080番) を許可
sudo ufw allow 8080/tcp 
## 4. 有効化
sudo ufw enable 
## 5. 確認
sudo ufw status 
4. プロジェクトディレクトリの作成 
mkdir /web-db-project
cd /web-db-project
5. Web 公開用ファイル (DocumentRoot) の準備 
mkdir htdocs

nano htdocs/index.html

ファイル内容: htdocs/index.html 

< !DOCTYPE html>
< html>
< head>
< title>Web/DB Test Page</>
</>
< body>
< h1>Docker Compose Web DB (Raspberry Pi)</>
< p>This page is served by the 'web' container on port 8080. </>
</>
</>

6. Docker Compose 構成ファイル (compose.yaml) の作成Web サーバー (httpd:latest) とデータベース (mysql:latest) を定義します 

## nano compose.yaml
ファイル内容: compose.yaml 

services: ##Web サーバー (Apache)

  web:

image: httpd:latest ## Docker Hub から最新の Apache イメージを使用

container_name: apache-server-rpi

    ports:
      "8080:80" ## ホストの 8080 ポートをコンテナの 80 ポートにマッピング
    volumes:
      ## ホストの htdocs フォルダをコンテナの公開フォルダにマウント
      -./htdocs:/usr/local/apache2/htdocs/
restart: always
  ##データベースサーバー (MySQL)
  db:
image: mysql:latest ## Docker Hub から最新の MySQL イメージを使用
container_name: mysql-db-rpi
    environment:
      ## 環境変数で初期設定 (秘密情報だが、ここではテスト値を使用)
    MYSQL_ROOT_PASSWORD: secure_root_password
    MYSQL_DATABASE: my_web_db
    MYSQL_USER: my_user
    MYSQL_PASSWORD: secure_user_password
    volumes:
      ## データベースデータを永続化するボリュームを定義
      -db_data:/var/lib/mysql
restart: always
##データの永続化用ボリューム定義 (ホストの /var/lib/docker/volumes/ に保存される)
volumes:
db_data:

7. Docker Compose の実行 19設定ファイルに基づき、コンテナ群を作成し、バックグラウンド (-d) で起動します docker compose up -d 

## 4. 動作確認と検証
コンテナの起動状態確認 
すべてのサービス (web, db) が Up 状態であることを確認します
docker compose ps
Web アクセス確認 (完成条件1の検証) 
クライアント PC のブラウザからアクセスします(Raspberry Pi の IP アドレスは 192.168.1.50 と仮定 27)

http://192.168.1.50:8080 にアクセス 

期待される結果: 「Docker Compose Web + DB (Raspberry Pi)」 と書かれたページが表示される 

3. ログ確認

Web コンテナのアクセスログを確認し、アクセスが記録されて
いることを検証します

docker compose logs web

期待される出力例 (末尾) apache-server-rpi 192.168.1.1 [07/Dec/2025:12:00:00 +0000] "GET / HTTP/1.1" 200 206

4. 永続化ボリュームの確認 (完成条件3の検証) 33ボリュームが作成されたことを確認します

docker volume ls

## トラブルシューティング

失敗例
- ブラウザからアクセスできない

原因

- ファイアウォール (UFW) で 8080番ポートが許可されていない

対処法

- sudo ufw allow 8080/tcp を実行し、sudo ufw reload または sudo ufw enable で反映する

失敗例

- docker compose up -d が失敗する

原因

- compose.yaml のインデントが間違っている (Tab 文字の使用など)

対処法

- YAML ファイルのインデントをすべてスペース2つに修正する

失敗例

- docker ps の STATUS が Exited になっている

原因

- コンテナ内のメインプロセスがすぐに終了した。特にDBコンテナの場合、環境変数 (パスワードなど) の設定ミスが多い 。

対処法

- docker compose logs db でログを確認し、エラーメッセージから原因を特定する
## 6. 参考資料 
Docker 公式ドキュメント: https://docs.docker.com/ 
Apache HTTP Server 公式ドキュメント: https://httpd.apache.org/ 
MySQL Docker Hub ページ: https://hub.docker.com/_/mysql 
Ubuntu UFW マニュアル: https://help.ubuntu.com/community/UFW