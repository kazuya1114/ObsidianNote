# ヒアドキュメントを使用してSQLファイルを使わずに複数のSQL文を実行する

```bash
psql -d {DB名} -U {ユーザ名} <<EOF
	BEGIN;
	
	CREATE TABLE test1 (
		id SERIAL PRIMARY KEY,
		name TEXT
	);

	INSERT INTO test1 (name) VALUES ('Alice');
	
	COMMINT;
EOF
```

# ログ出力

- `>> ログファイルパス`：上書き
- `> ログファイルパス`：ファイル内容の置き換え
- `2>> ログファイルパス`：エラー内容のみを出力し、ファイル内容は上書き
- `2> ログファイルパス`：エラー内容のみ出力し、ファイル内容は置き換え

```bash
LOG_FILE="/logs/log.txt"

echo "$(date "+%Y-%m-%d %H:%M:%S") [INFO]処理開始" >> $LOG_FILE
echo "$(date "+%Y-%m-%d %H:%M:%S") [ERROR]実行エラー" >> $LOG_FILE
echo "$(date "+%Y-%m-%d %H:%M:%S") [INFO]処理終了" >> $LOG_FILE

psql -d db_name -U user_name -f example.sql 2> $LOG_FILE

# エラー内容一行ずつにタイムスタンプとログレベルを付与する
# awkは渡ってきたテキストを行ごとに処理する
psql -d db_name -U user_name -f example.sql 2> >(awk '{
	print strftime("%Y-%m-%d %H:%M:%S"), "[INFO]", $0
}' >> $LOG_FILE)
```

# ループ処理

### 基本

```bash
while IFS='{区切り文字}' read -r {ループ内で使う変数名}; do
  #処理
done
```

### ファイルから読み取った値を使用してループ処理

```bash
while IFS'{区切り文字}' read -r {ループ内で使う変数名}; do
  #処理
done < "${ファイルパスを記述した変数}"
```

### テキストから読み取った値を使用してループ処理

```bash
while IFS'{区切り文字}' read -r {ループ内で使う変数名}; do
  #処理
done <<< "${テキスト}"
```

# コマンドの実行結果（標準出力内容）を変数に格納する

```bash
{変数}=$({コマンド})

# 例
result=$(ls)
```

# 直前に使用したコマンドの終了ステータス（exit code）を参照する

```bash
$?

# 使い方
if [$? -eq 0]; then
	# 直前のコマンドが正常終了の場合
else
	# 直前のコマンドが正常終了以外の場合
fi
```

# 別bashファイルをimportし、そのファイルで定義された関数を使用する

### 共通スクリプト（Common.bash）

```bash
#!/bin/bash

: <<'END'
@author Kazuya, Hata
@version 1.0
END

: <<'END'
$3で指定されたファイルに$2で指定された内容をログとして吐き出す
@param $1 処理結果(0：正常、0以外：エラー)
@param $2 ログの出力内容
@param $3 リダイレクト先のログファイルパス
END
outputLog() {
  if [ $1 -eq 0]; then
    # 出力内容がある場合のみ、ログ出力を行う
    if [[ -n $2 ]]; then
      echo "$(date "+%Y-%m-%d %H:%M:%S") [INFO]$2" >> $3
    fi
  else
    echo "$(date "+%Y-%m-%d %H:%M:%S") [ERROR]$2" >> $3
  fi
}
```

### 共通スクリプトをimportし、関数を使用する

```bash
#!/bin/bash

# 指定されたスクリプトを読み込む
source ./Common.bash

# 読み込んだ関数の使用(引数は半角空白で区切る)
outputLog $? "テストです。" "/logs/log.txt"
```

# データの差替えを行う

データ投入の方法について、単純なデータの差替えの場合、TRUNCATE実行後にINSERT SELECT、COPY、\copyといった方法があるが、シェル等を使ってDBサーバ上で直接作業をする場合、`INSERT SELECTが一番パフォーマンスがいい`
※COPY系は別サーバへ情報を書き出す場合などに有利

```bash
LOG_FILE=/tmp/db_sync.log
PASSWORD=postgres
USER=postgres
DBNAME=testdb
TARGET_TABLE=test_table
SOURCE_TABLE=fdw_table

# データ同期処理
# 同期先テーブル（内部テーブルを指定）
# 同期元テーブル（外部テーブルを指定）
function data_sync() {
	local sync_target_table=$1
	local sync_source_table=$2

	/usr/sbin/runuser -l postgres -c "
		# DB接続用のパスワードを設定
		export PGPASSWORD=$PASSWORD

		# -v ON_ERROR_STOP=1：エラーが起きた時点で処理を中断
		psql -U $USER -d DBNAME -v ON_ERROR_STOP=1 <<EOF
			BEGIN;
			TRUNCATE TABLE $sync_target_table;
			INSERT INTO $sync_target_table SELECT * FROM $sync_source_table;
			COMMIT;
EOF
" 2>> "$LOG_FILE"
}

data_sync "$TARGET_TABLE" "$SOURCE_TABLE"
```
