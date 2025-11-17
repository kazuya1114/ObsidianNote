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

```bash
LOG_FILE="/logs/log.txt"

echo "$(date "+%Y-%m-%d %H:%M:%S") [INFO]処理開始" >> $LOG_FILE
echo "$(date "+%Y-%m-%d %H:%M:%S") [ERROR]実行エラー" >> $LOG_FILE
echo "$(date "+%Y-%m-%d %H:%M:%S") [INFO]処理終了" >> $LOG_FILE
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

