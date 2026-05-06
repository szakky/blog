---
title : "rodoの技術的な話"
date : 2026-05-06
draft : false
---

# 目次
- [はじめに](#はじめに)
- [使用した技術スタック一覧](#使用した技術スタック一覧)
- [アプリケーション](#アプリケーション)
    - [ディレクトリ構成](#ディレクトリ構成)
    - [golang](#golang)
        - [1.ルーティング(main.go)](#1-ルーティングmaingo)
        - [2.アプリケーション機能(handler.go)](#2-アプリケーション機能-handlergo)
        - [3.データベース(mysql.go)](#3データベースmysqlgo)
- [サービス基盤](#サービス基盤)
    - [Docker](#docker)
    - [kubernetes](#kubernetes)
- [おわりに](#おわりに)

# はじめに

前回の[ルーム型To-Doアプリを作ってみた]の技術スタックについてまとめていきます。
  
以下はRodoのgithubリポジトリになります。

[https://github.com/szakky/rodo](https://github.com/szakky/rodo)

# 使用した技術スタック一覧

使用言語: Go, HTML

DB: MySQL

インフラ: docker ,kubernetes ,cloudflare

![フローチャート](/blog/images/rodo-flow.png)


# アプリケーション
## ディレクトリ構成
```
├── db/
│   └── mysql.go
├── templates/
│   ├── room.html
│   └── top.html
├── docker-compose.yml
├── dockerfile
├── handler.go
└── main.go
```
## golang
### 1. ルーティング(main.go)

ここでは、アプリケーション機能をどのようにしてルーティングしているか説明していきます。
```Go
package main

import (
	"database/sql"
	"log"
	"net/http"
	"todo-api/db"

	_ "github.com/go-sql-driver/mysql"
)

var conn *sql.DB

func main() {
	var err error
	conn, err = db.Connect()
	if err != nil {
		log.Fatal("db error:", err)
	}
	defer conn.Close() //kill db connect

	if err = conn.Ping(); err != nil {
		log.Fatal("db error:", err)
	}
	log.Println("db connected")
	log.Println("ready")

	http.HandleFunc("/", topPage)
	http.HandleFunc("/room/", roomPage)
	http.HandleFunc("/add", add)
	http.HandleFunc("/update", updateTask)
	http.HandleFunc("/delete", deleteTask)
	http.HandleFunc("/delete-all", deleteAll)
	log.Println("waiting for requests...")
	http.ListenAndServe(":8080", nil)
}
```
`todo-api/db` パッケージは次の`mysql.go` をインポートしています。グローバル変数`conn` は、`*sql.DB` dbの接続情報が入っています。`todo-api/db` パッケージを使って、データベースと接続し、結果として接続情報（`conn`）と、エラー（`err`）を受け取ります。`db.Connect()` が成功しても、ネットワークの問題で通信ができないこともあるので、`conn.Ping()` で通信ができるかチェックします。`http.HandleFunc()`は、URLのパスに対して、どの関数を実行させるか設定しています。これで`handler.go` のアプリケーション機能を呼び出しています。そして、`http.ListenAndServe()` でポート番号`8080` 番のサーバーを起動させ、アクセスを待つ状態にします。
### 2. アプリケーション機能 (handler.go)

`handler.go` ではタスクの追加やタグ、メモのような機能をまとめています。

/`toppage()` 
```Go
func topPage(w http.ResponseWriter, r *http.Request) {
	tmpl, err := template.ParseFiles("templates/top.html")
	if err != nil {
		http.Error(w, "parse Error", http.StatusInternalServerError)
		log.Println("parse error:", err)
		return
	}

	err = tmpl.Execute(w, nil)
	if err != nil {
		http.Error(w, "execute Error", http.StatusInternalServerError)
		log.Println("execute error:", err)
		return
	}
}
```
`toppage()` は、`templates/top.html` を読み込み、ブラウザに返してトップページを表示します。エラーが発生した場合は、ブラウザにエラーメッセージを表示し、ログに詳細を記録します。

/`roomPage()` 
```Go
type TaskView struct {
	ID         int
	Title      string
	Categorize string
	Memo       string
	TagColor   string
}

func roomPage(w http.ResponseWriter, r *http.Request) {

	jst := time.FixedZone("JST", 9*60*60)
	todayStr := time.Now().In(jst).Format("2006-01-02")
	roomID := r.URL.Query().Get("room_id")
	
	rows, err := conn.Query("SELECT id, title, categorize, COALESCE(memo, '') FROM tasks WHERE DATE(created_at)= ? AND room_id = ?", todayStr, roomID) 
	if err != nil {
		http.Error(w, "DB Error", http.StatusInternalServerError)
		return
	}
	defer rows.Close() //kill db connect

	var tasks []TaskView
	for rows.Next() {
		var t TaskView
		if err := rows.Scan(&t.ID, &t.Title, &t.Categorize, &t.Memo); err != nil {
			http.Error(w, "DB Error", http.StatusInternalServerError)
			return
		}

		if t.Categorize != "" {
			t.TagColor = getColorForTag(t.Categorize)
		}

		tasks = append(tasks, t)
	}

	tmpl, err := template.ParseFiles("templates/room.html")
	if err != nil {
		http.Error(w, "template parse error", http.StatusInternalServerError)
		log.Println("template parse error:", err)
		return
	}

	data := struct {
		Tasks  []TaskView
		RoomID string
	}{
		Tasks:  tasks,
		RoomID: roomID,
	}

	if err := tmpl.Execute(w, data); err != nil {
		http.Error(w, "template execute error", http.StatusInternalServerError)
		log.Println("template execute error:", err)
		return
	}
}

```
 `roomPage()` 関数では、その日限定の `room_id` に紐づくタスクをデータベースから取得し、tasks配列に格納しています。データ取得後は `templates/room.html` を読み込み、ルームページを表示します。エラーが発生した場合は、ブラウザにエラーメッセージを表示し、ログに詳細を記録します。

また、コメントでも書いていますが、defer rows.Close() と書いておくことで、その後のfor文でエラーが起きても、問題なく処理が完了しても、関数から抜けるときにgolang側で自動的にデータベースとの接続を切ってくれます。DBコネクションの解放漏れはシステムに悪影響を及ぼす原因になりやすいため、安全な設計を心がけて実装しました。

/`add()` 
```Go
func add(w http.ResponseWriter, r *http.Request) {
	title := r.URL.Query().Get("title")
	categorize := r.URL.Query().Get("categorize")
	memo := r.URL.Query().Get("memo")
	roomID := r.URL.Query().Get("room_id")

	_, err := conn.Exec("INSERT INTO tasks (title, categorize, memo, room_id) VALUES (?, ?, ?, ?)", title, categorize, memo, roomID)
	if err != nil {
		log.Printf("Added failed: %v\n", err)
		http.Error(w, "Added failed", http.StatusInternalServerError)
		return
	}

	if roomID == "" {
		http.Redirect(w, r, "/", http.StatusSeeOther)
		return
	}

	http.Redirect(w, r, "/room/?room_id="+roomID, http.StatusSeeOther)
}
```
`add()` では、URLで入力された `title/categorize/memo/room_id` を取得し、dbに追加します。room_idがなければ `/` (トップページ)へ、room_idがあれば`/room/?room_id=roomID`(ルームページ)のURLページになります。以下の`updateTask() deleteTask() deleteAll()` は全てルーム内の機能なので `http.Redirect(w, r, "/room/?room_id="+roomID, http.StatusSeeOther)` があります。

/`updateTask()` 
```Go
func updateTask(w http.ResponseWriter, r *http.Request) {
	idStr := r.URL.Query().Get("id")
	memo := r.URL.Query().Get("memo")
	categorize := r.URL.Query().Get("categorize")
	roomID := r.URL.Query().Get("room_id")

	id, err := strconv.Atoi(idStr)
	if err != nil {
		http.Error(w, "Invalid ID", http.StatusBadRequest)
		return
	}

	_, err = conn.Exec("UPDATE tasks SET memo = ?, categorize = ? WHERE id = ? AND room_id = ?", memo, categorize, id, roomID)
	if err != nil {
		http.Error(w, "failed to update task", http.StatusInternalServerError)
		return
	}

	http.Redirect(w, r, "/room/?room_id="+roomID, http.StatusSeeOther)
}
```
`updateTask()`関数では、URLで入力された `title/categorize/memo/room_id` を取得し、idStrは文字列なので`strconv.Atoi(idStr)` とすることでint型に変換できます。変換できなかった場合、エラー表示がされます。 SQL文では `WHERE id = ? AND room_id = ?"` idとroom_idが同じならタスクを更新するようになります。一致しなかった場合、エラー表示されます。

/`deleteTask()` 
```Go
func deleteTask(w http.ResponseWriter, r *http.Request) {
	idStr := r.URL.Query().Get("id")
	roomID := r.URL.Query().Get("room_id")

	id, err := strconv.Atoi(idStr)
	if err != nil {
		http.Error(w, "Invalid ID", http.StatusBadRequest)
		return
	}

	_, err = conn.Exec("DELETE FROM tasks WHERE id = ? AND room_id = ?", id, roomID)
	if err != nil {
		http.Error(w, "failed to delete task", http.StatusInternalServerError)
		return
	}

	http.Redirect(w, r, "/room/?room_id="+roomID, http.StatusSeeOther)
}
```
`deleteTask()` 関数は、先ほどの`updateTask()` と似ており `id/room_id` だけを取得し、`("DELETE FROM tasks WHERE id = ? AND room_id = ?"` idとroom_idが同じならそのタスクを削除します。

/`deleteAll()` 
```Go
func deleteAll(w http.ResponseWriter, r *http.Request) {
	roomID := r.URL.Query().Get("room_id")

	_, err := conn.Exec("DELETE FROM tasks WHERE room_id = ?", roomID)
	if err != nil {
		log.Printf("Delete failed: %v\n", err)
		http.Error(w, "Delete failed", http.StatusInternalServerError)
		return
	}
	http.Redirect(w, r, "/room/?room_id="+roomID, http.StatusSeeOther)
}
```
`deleteAll()` 関数では、`room_id` だけ取得し、指定のroom_idのすべてのタスクを削除します。

### 3.データベース(mysql.go)
```Go
func Connect() (*sql.DB, error) {
	dbUser := os.Getenv("DB_USER")
	dbPass := os.Getenv("DB_PASSWORD")
	dbHost := os.Getenv("DB_HOST")
	dbPort := os.Getenv("DB_PORT")
	dbName := os.Getenv("DB_NAME")

	if dbUser == "" {
		return nil, fmt.Errorf("error: DB_USER is not set")
	}

	dsn := fmt.Sprintf("%s:%s@tcp(%s:%s)/%s?parseTime=true", dbUser, dbPass, dbHost, dbPort, dbName)

	db, err := sql.Open("mysql", dsn)
	if err != nil {
		return nil, err
	}

	if err = db.Ping(); err != nil {
		return nil, err
	}

	todoTableSQL := `
	CREATE TABLE IF NOT EXISTS tasks (
		id INT AUTO_INCREMENT PRIMARY KEY,
		title VARCHAR(255) NOT NULL,
		categorize VARCHAR(255) NOT NULL,
		memo TEXT,
		room_id VARCHAR(255) NOT NULL,
		created_at DATETIME DEFAULT CURRENT_TIMESTAMP
	);`

	_, err = db.Exec(todoTableSQL)
	if err != nil {
		return nil, err
	}

	return db, nil
}
```
ユーザー名やパスワードなどの情報をコードにベタ書きせず、`os.Getenv` で環境変数から読み込んでいます。テーブルの部分では、アプリケーションを初めて起動したときに自動でテーブル作成しています。`db.Exec(todoTableSQL)` で実際にデータベースに送信して実行します。最後に、`db` 情報と、エラーなしを`main` 関数に返します。

ここまでが主な実装になります。ここからは、サービス基盤を使いデプロイしているか説明していきます。
# サービス基盤

サービス基盤では、Dockerとkubernetes、Cloudflareを使用しました。
## Docker

`docker-compose.yml` 
```yml
services:
  app:
    build: .
    restart: always
    ports:
      - "8080:8080"
    env_file:
      - .env
    depends_on:
      - db

  db:
    image: mysql:8.0
    ports:
      - "3307:3306"
    env_file:
      - .env
    volumes:
      - db-data:/var/lib/mysql

volumes:
  db-data:
```
自分のPCのローカル環境にすでに別のMySQLがインストールされていて3306ポートを使っているので、競合して起動エラーになってしまいます。そのため、ホスト側は `3307` にズラし、コンテナ側の `3306` に繋ぐように設定しています。

`env_file: - .env`は、 `mysql.go`で環境変数`os.Getenv`で取得した情報を、`.env` という外部ファイルから読み込みます。コードの中に直接パスワードを書かなくて済むため、セキュリティ上重要な設定になります。

`dockerfile` は以下のようになります。
```
FROM golang:1.24.4-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -o todo-api .

FROM alpine:latest

WORKDIR /app

COPY --from=builder /app/todo-api .

COPY --from=builder /app/templates ./templates

EXPOSE 8080

CMD ["./todo-api"]
```

補足: `template/.html` は`//go:embed templates/`を使えば`go` ファイルと読み込まれ `COPY --from=builder /app/templates ./templates` は不要になります。(今後、修正予定。)
## Kubernetes
`todo-api.yaml` 
```pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-api-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: todo-api
  template:
    metadata:
      labels:
        app: todo-api
    spec:
      containers:
      - name: todo-api
        image: ennyou/todo-api:v3
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: "db"
        - name: DB_PORT
          value: "3306"
        - name: DB_USER
          value: "root"
        - name: DB_PASSWORD
          value: "secret_password"
        - name: DB_NAME
          value: "todo_app"
---
apiVersion: v1
kind: Service
metadata:
  name: todo-api-service
spec:
  type: ClusterIP
  selector:
    app: todo-api
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```
コンテナ起動設定のDeploymentと通信を受け取るServiceを定義しています。`Deployment`では、次の`mysql.pod`に接続します。`Service`でClusterIP という設定により、セキュリティ上クラスタ内部からしかアクセスできない状態にしています。

`mysql.pod`
```pod
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:8.0
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "secret_password"
        - name: MYSQL_DATABASE
          value: "todo_app"
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
```
`service`で`todo-api.yaml`と接続をします。`Deployment`で環境変数`MYSQL_DATABASE`で、初回起動時に自動的にデータベースを作成してくれます。`PersistentVolumeClaim`はMySQLのデータ保存先（/var/lib/mysql）にマウントさせることで、コンテナが作り直されても過去のデータがしっかりと引き継がれます。

`cloudflare.pod` 
```pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflare-tunnel-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudflare-tunnel
  template:
    metadata:
      labels:
        app: cloudflare-tunnel
    spec:
      containers:
      - name: cloudflared
        image: cloudflare/cloudflared:latest
        args:
        - tunnel
        - --no-autoupdate
        - run
        - --token
```
今回は Cloudflare Tunnel（Zero Trust） を採用しています。cloudflared コンテナが、Kubernetesの内側から外側（Cloudflare側）に向かって安全なトンネルを掘ります。サーバー側に外部からの通信を受け入れるトンネルを開ける必要がないため、悪意のある直接攻撃（DDoSやポートスキャン）を物理的に防ぐことができます。

(※実際に稼働させる際は、--token の後に、Cloudflareダッシュボードで発行した認証トークンを設定します。)

# おわりに
今回は、rodoの技術スタックについてまとめてみました。KubernetesやCloudflareなど、これまであまり触れたことのなかった技術にも挑戦することができ、インフラからアプリケーション層まで非常に有意義な勉強になりました。今後はこの知見を活かし、さらなる機能追加や別のアプリケーション開発にも取り組んでいきたいです。