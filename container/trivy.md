# Trivy

Trivy を使ってローカルにある Docker イメージに脆弱性スキャンを実行する手順。 macOS 上で手軽に検証するなら公式の Docker イメージを使うと良い。

```sh
docker pull aquasec/trivy:latest

docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $HOME/Library/Caches:/root/.cache/ \
  aquasec/trivy IMAGE_NAME
```

ローカルの Docker イメージをスキャンするには、ホスト側の socket ファイルをマウントする必要がある。スキャン結果は次のような感じ。

```text
hello-world (alpine 3.11.3)
===========================
Total: 1 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 1, CRITICAL: 0)

+---------+------------------+----------+-------------------+---------------+--------------------------------+
| LIBRARY | VULNERABILITY ID | SEVERITY | INSTALLED VERSION | FIXED VERSION |             TITLE              |
+---------+------------------+----------+-------------------+---------------+--------------------------------+
| openssl | CVE-2020-1967    | HIGH     | 1.1.1d-r3         | 1.1.1g-r0     | openssl: Segmentation fault in |
|         |                  |          |                   |               | SSL_check_chain causes denial  |
|         |                  |          |                   |               | of service                     |
+---------+------------------+----------+-------------------+---------------+--------------------------------+
```

## CI に組み込む際の注意点

### 終了ステータスを変更する

脆弱性が見つかった場合に CI を失敗させたい場合は `--exit-code 1` のオプションを付ける必要がある。オプションを付けなければ `0` が返るので CI は通ってしまう。

`CRITICAL` のみ検出して CI を失敗させることもできる。

```sh
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $HOME/Library/Caches:/root/.cache/ \
  aquasec/trivy --exit-code 1 --severity CRITICAL alpine:3.11
```

### 結果を JSON ファイルに保存する

標準出力だとあとから参照するときに困るので、ファイルに出力して S3 などに集めておきたい。そういうときは `-f json` と `-o <file>` のオプションを付ければファイルに出力することができる。

```sh
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $HOME/Library/Caches:/root/.cache/ \
  aquasec/trivy -f json -o /root/.cache/trivy/results.json alpine:3.11
```

### 脆弱性データベースのキャッシュ

初回実行時は脆弱性のデータベースをダウンロードしてくるため時間がかかる。キャッシュディレクトリをマウントすれば次回からは差分のみダウンロードしてくれる。

CI に組み込むときは、このファイルをキャッシュして使い回す工夫が必要。

```sh
$ tree $HOME/Library/Caches/trivy
/Users/sakai-manabu/Library/Caches/trivy
├── db
│   ├── metadata.json
│   └── trivy.db
└── fanal
    └── fanal.db

2 directories, 3 files
```
