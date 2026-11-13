# AWS公式 AWS Emulator 導入・実行ガイド

AWS公式の **AWS Emulator** をローカルPCに導入し、AWS CLIからS3バケットを作成・操作するまでの手順です。

- **想定環境**：Windows 11、Docker Desktop、PowerShell、AWS CLI v2
- **ゴール**：S3バケットの作成、一覧取得、オブジェクトのアップロード・ダウンロード・削除
- **注意**：エミュレーターは実際のAWS環境ではありません。APIの再現範囲に制限があります。

## 1. 構成

```text
Windows 11
├── PowerShell
│   └── AWS CLI ───────────┐
└── Docker Desktop         │ HTTPS (127.0.0.1:9297)
    └── AWS Emulator ◀─────┘
        └── S3 APIを模倣
            └── ローカルのS3バケット／オブジェクト
```

AWS CLIの接続先をローカルのAWS Emulatorに変更して操作します。AWSアカウントは不要です。

## 2. 前提ソフトウェアの準備

| ソフトウェア | 用途 | 入手先 |
|---|---|---|
| Docker Desktop | AWS Emulatorの実行 | [Docker公式](https://www.docker.com/products/docker-desktop/) |
| AWS CLI v2 | AWS APIの操作 | [AWS CLIインストールガイド](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) |

### Step 1. Docker Desktopを確認

Docker Desktopをインストールし、起動してからPowerShellで確認します。

```powershell
docker --version
docker info
```

`docker info` でサーバー情報が表示されれば、Docker Engineに接続できています。

### Step 2. AWS CLIを確認

AWS CLI v2をインストール後、PowerShellで実行します。

```powershell
aws --version
```

## 3. AWS EmulatorのDockerイメージをビルド

AWS公式のDockerfileを利用してイメージを作成します。以下は **x86_64** 用です。ARM64の場合はRPMのアーキテクチャを変更してください。

### Step 3. 作業ディレクトリを作成

```powershell
mkdir aws-emulator-demo
cd aws-emulator-demo
```

### Step 4. Dockerfileを作成

作業ディレクトリに、拡張子なしの `Dockerfile` を作成し、以下を保存します。

```dockerfile
FROM public.ecr.aws/amazonlinux/amazonlinux:2023-minimal

ADD https://downloads.aws.com/aws-emulator/packages/aws-emulator/LTS/latest/aws-emulator-x86_64.rpm /tmp/aws-emulator.rpm

RUN rpm -i /tmp/aws-emulator.rpm && rm /tmp/aws-emulator.rpm \
    && mkdir -p /home/emulator && chown 1000 /home/emulator

USER 1000
ENV HOME=/home/emulator

EXPOSE 9297

ENTRYPOINT ["aws-emulator", "-b", "0.0.0.0"]
```

| Dockerfile命令 | 役割 |
|---|---|
| `FROM` | Amazon Linux 2023を使用 |
| `ADD` | AWS Emulator RPMを取得 |
| `RUN` | RPMのインストールと作業ディレクトリ作成 |
| `USER` | 非rootユーザーで実行 |
| `EXPOSE` | 9297番ポートを宣言 |
| `ENTRYPOINT` | エミュレーターを起動 |

### Step 5. Dockerイメージをビルド

```powershell
docker build -t aws-emulator .
docker images aws-emulator
```

`aws-emulator` イメージが表示されれば完了です。

## 4. AWS Emulatorを起動

### Step 6. 認証情報の共有ディレクトリを作成

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.aws\emulator"
```

エミュレーターが生成する認証情報とHTTPS証明書を、Windows側から読み取るために使用します。

### Step 7. コンテナを起動

```powershell
$port = 9297

docker run --rm --name aws-emulator `
  --memory=2g --memory-swap=2g `
  -p "127.0.0.1:${port}:${port}" `
  -v "$env:USERPROFILE\.aws\emulator:/home/emulator/.aws/emulator" `
  aws-emulator -p $port
```

このPowerShellウィンドウは開いたままにしてください。

- `127.0.0.1` へのバインドにより、ホストPC以外からのアクセスを制限します。
- `--rm` は停止時にコンテナを削除します。
- エミュレーター内のリソースはメモリ上で管理され、停止すると失われることがあります。永続的なAWS環境の代わりにはなりません。

## 5. AWS CLIの接続先を設定

**別のPowerShellウィンドウ**を開き、以降のコマンドを実行します。

### Step 8. 環境変数を設定

```powershell
$env:AWS_ENDPOINT_URL = "https://127.0.0.1:9297"
$env:AWS_SHARED_CREDENTIALS_FILE = "$env:USERPROFILE\.aws\emulator\9297\emulator_credentials"
$env:AWS_CONFIG_FILE = "$env:USERPROFILE\.aws\emulator\9297\config"
$env:AWS_CA_BUNDLE = "$env:USERPROFILE\.aws\emulator\9297\ca.pem"
$env:AWS_EC2_METADATA_DISABLED = "true"
$env:AWS_DISABLE_HOST_PREFIX_INJECTION = "true"
$env:AWS_REGION = "us-east-1"
$env:AWS_DEFAULT_REGION = "us-east-1"

Remove-Item Env:AWS_ACCESS_KEY_ID, Env:AWS_SECRET_ACCESS_KEY, Env:AWS_SESSION_TOKEN, Env:AWS_PROFILE -ErrorAction SilentlyContinue
```

重要な設定：

| 環境変数 | 目的 |
|---|---|
| `AWS_ENDPOINT_URL` | AWS CLIのAPI送信先をローカルに変更 |
| `AWS_SHARED_CREDENTIALS_FILE` | エミュレーター専用の認証情報を使用 |
| `AWS_CONFIG_FILE` | エミュレーター用AWS設定ファイルを参照 |
| `AWS_CA_BUNDLE` | エミュレーターのHTTPS証明書を信頼 |

> **誤接続防止**：この環境変数を設定していない別のシェルから `aws` を実行すると、実際のAWSに接続する場合があります。操作前に送信先を必ず確認してください。

### Step 9. 接続を確認

ヘルスチェック：

```powershell
curl.exe --cacert "$env:AWS_CA_BUNDLE" https://127.0.0.1:9297/__admin/health
```

正常時の応答例：

```text
ok
```

STS APIの確認：

```powershell
aws sts get-caller-identity
```

想定される出力例（値は環境によって異なります）：

```json
{
  "UserId": "EMULATOR...",
  "Account": "000000000000",
  "Arn": "arn:aws:iam::000000000000:user/..."
}
```

## 6. S3バケットを作成

### Step 10. バケットを作成

```powershell
aws s3 mb s3://local-getting-started
```

出力例：

```text
make_bucket: local-getting-started
```

### Step 11. バケット一覧を表示

```powershell
aws s3 ls
```

出力例：

```text
2026-11-13 14:00:00 local-getting-started
```

## 7. オブジェクトを操作

### Step 12. テストファイルを作成

```powershell
Set-Content -Path hello.txt -Value "Hello from AWS Emulator" -Encoding utf8
```

### Step 13. アップロード

```powershell
aws s3 cp hello.txt s3://local-getting-started/
```

### Step 14. オブジェクト一覧を取得

```powershell
aws s3 ls s3://local-getting-started/
```

`hello.txt` が表示されればアップロード成功です。

### Step 15. ダウンロード・内容確認

```powershell
aws s3 cp s3://local-getting-started/hello.txt downloaded.txt
Get-Content downloaded.txt
```

期待される内容：

```text
Hello from AWS Emulator
```

## 8. リソースを削除

### Step 16. オブジェクトを削除

```powershell
aws s3 rm s3://local-getting-started/hello.txt
```

### Step 17. バケットを削除

```powershell
aws s3 rb s3://local-getting-started
aws s3 ls
```

バケット一覧に表示されなくなれば削除成功です。

## 9. コンテナを停止

起動用PowerShellウィンドウで `Ctrl+C` を押すか、別のPowerShellから停止します。

```powershell
docker stop aws-emulator
```

実際のAWSを操作する際は、新しいPowerShellセッションを開き、接続先・認証情報を確認してください。

## 10. トラブルシューティング

| エラー・症状 | 主な確認事項 |
|---|---|
| `docker` が認識されない | Docker DesktopのインストールとPATH |
| Docker daemonに接続できない | Docker Desktopの起動状態 |
| `Connection refused` | コンテナの状態、9297番ポート |
| `SSL certificate verify failed` | `AWS_CA_BUNDLE` のパスと証明書 |
| `InvalidAccessKeyId` | 認証情報ファイルとAWS環境変数 |
| `NotImplemented` / HTTP 501 | 該当APIの対応状況 |
| 終了コード137 | Dockerの利用可能メモリ |

## 11. AWS CLIコマンドとAPIの対応

| AWS CLI操作 | 主なAWS API |
|---|---|
| `aws s3 mb` | `CreateBucket` |
| `aws s3 ls` | `ListBuckets` / `ListObjectsV2` |
| `aws s3 cp`（アップロード） | `PutObject` |
| `aws s3 cp`（ダウンロード） | `GetObject` |
| `aws s3 rm` | `DeleteObject` |
| `aws s3 rb` | `DeleteBucket` |

> AWS CLIは条件によって追加のAPIを呼び出す場合があります。また、AWS Emulatorでの成功は、実際のAWSにおけるIAM権限・ネットワーク挙動・サービス連携の成功を保証しません。

## 12. 参考資料

- [AWS Emulator公式ドキュメント](https://docs.aws.amazon.com/emulator/)
- [DockerでAWS Emulatorを起動](https://docs.aws.amazon.com/emulator/latest/userguide/getting-started-docker.html)
- [AWS CLI・SDKの接続先設定](https://docs.aws.amazon.com/emulator/latest/userguide/point-sdk-to-emulator.html)
- [AWS Emulatorのセキュリティ](https://docs.aws.amazon.com/emulator/latest/userguide/security.html)

---

**次の検証候補**：CloudFormationテンプレートからのS3バケット作成。ただし、AWS EmulatorのCloudFormation API対応範囲を事前に確認してください。
