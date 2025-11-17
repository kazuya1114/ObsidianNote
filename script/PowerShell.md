# コメント
```ps1
# 通常のコメント

<#
行コメント
@author
@version
#>
```

# 変数の定義
```ps1
# $変数名 = 値
$num = 1
```

# if文
```ps1
<#
if (条件) {
  # 条件がtrueの際の処理
} else {
  # 条件がfalseの際の処理
}
#>

if ($i = 0) {
  Write-Output "OK"
} else {
  Write-Output "NO"
}
```

# for文
```ps1
<#
foreach (ループ行を扱う変数名 in ループ対象) {
  # 処理
}
#>

foreach ($row in $data) {
  $i = $row
}
```

# 環境変数の定義
```ps1
$env:URL = "https://test.co.jp/"
```

# 関数の定義と呼び出し
```ps1
<#
ログ出力処理
@param $flg true：INFOログを出力、false：ERRORログを出力
@param $content 出力内容
@param $logFilePath 出力先ファイルパス
#>
function writeLog($flg, $content, $logFilePath) {
  # 出力先のファイル・フォルダが存在しない場合は、作成する
  if (!(Test-Path -Path $logFilePath)) {
    New-Item -ItemType Directory -Path $logFilePath | Out-Null
  }

  # ログ出力処理
  $currentTime = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
  if ($flg -eq $true) {
    $output = $currentTime + " [INFO] " + $content
    Add-Content -Path $logFilePath -Value $output -Encoding UTF8
  } else {
    $output = $currentTime + " [ERROR] " + $content
    Add-Content -Path $logFilePath -Value $output -Encoding UTF8
  }
}

# 関数の呼び出し
writeLog $true "テスト1" "/var/test/logs/log.txt"
writeLog $false "テスト2" "/var/test/logs/log.txt"
```

# APIを呼び出す例(例題：MattermostのAPI)
### 共通処理を提供するCommon.ps1スクリプト(ここではログ出力)
```ps1
<#
ログ出力処理
@param $flg true：INFOログを出力、false：ERRORログを出力
@param $content 出力内容
@param $logFilePath 出力先ファイルパス
#>
function writeLog($flg, $content, $logFilePath) {
  # 出力先のファイル・フォルダが存在しない場合は、作成する
  if (!(Test-Path -Path $logFilePath)) {
    New-Item -ItemType Directory -Path $logFilePath | Out-Null
  }

  # ログ出力処理
  $currentTime = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
  if ($flg -eq $true) {
    $output = $currentTime + " [INFO] " + $content
    Add-Content -Path $logFilePath -Value $output -Encoding UTF8
  } else {
    $output = $currentTime + " [ERROR] " + $content
    Add-Content -Path $logFilePath -Value $output -Encoding UTF8
  }
}
```

### APIを実行するメインのスクリプト
```ps1
# ユーザ情報を格納するクラス
class User {
  [string]$id
  [string]$username
  [string]$first_name
  [string]$last_name

  User([string]$id, [string]$username, [string]$first_name, [string]$last_name) {
    $this.id = $id
    $this.username = $username
    $this.first_name = $first_name
    $this.last_name = $last_name
  }
}

# Common.ps1を読み込む
. ./Common.ps1

# ログインAPIのリクエストボディを実装
$request = @{
  login_id = "admin"
  password = "password"
} | ConvertTo-Json

# APIの実行
$LoginResponse = Invoke-WebRequest -Uri "https://mattermost.co.jp/api/v4/users/login" -Method Post -Body $request -ContentType "application/json"

# レスポンスからトークンを取得し、以降の処理の認証に使用する
$Token = $LoginResponse.Headers["Token"]

# httpヘッダーの作成
$headers = @{
  'Authorization' = "Bearer $Token"
}

# csvファイルから取得対象のユーザ名を読み取りAPIを発行する
# 本スクリプトと同じディレクトリにあるCSVを読み取る
$dir = Split-Path $MyInvocation.MyCommand.Path -Parent
$path = "$dir\test.csv"
$data = Import-Csv -Path $path -Encoding UTF8

# 取得したユーザデータを格納するリストを実装
$users = New-Object System.Collections.ArrayList

foreach ($row in $data) {
  try {
    $response = Invoke-WebRequest -Method Get -Uri "https://mattermost.co.jp/api/v4/users/username/$($row.username)" -Headers $headers -ContentType "application/json; charset=utf-8"

    # レスポンスをUTF-8にエンコード
    $reader = New-Object System.IO.StreamReader($response.RowContentStream, [System.Text.Encoding]::UTF8)
    $content = $reader.ReadToEnd()

    # レスポンスをJsonに変換
    $result = $content | ConvertFrom-Json

    # ユーザをリストに追加
    $users.Add([User]::new($result.id, $result.username, $result.first_name, $result.last_name)) | Out-Null
  } catch {
    # レスポンスがエラーの場合、詳細をログへ出力
    $errorMessage = #_.Exception.Response.GetResponseStream()
    $reader = New-Object System.IO.StreamReader($errorResponse)
    $reader.BaseStream.Position = 0
    $reader.DiscardBufferedData()
    $responseBody = $reader.ReadToEnd() | ConvertFrom-Json

    # エラー詳細を表示
    writeLog false $errorMessage "/var/test/logs/log.txt"
    writeLog false $responseBody.message "/var/test/logs/log.txt"
  }

  Write-Output $users
  writeLog true "処理終了" "/var/test/logs/log.txt"
}
```
