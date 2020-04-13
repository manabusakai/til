# Modify DB Instance

AWS CLI で RDS のインスタンスタイプを変更する手順を検証する。

変更する内容によってどういう影響があるのかは「[MySQL データベースエンジンを実行する DB インスタンスの変更](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/USER_ModifyInstance.MySQL.html)」を確認すること。ダウンタイムが発生しないケースでもパフォーマンス低下を引き起こすことがあるので要注意。

ほとんどの作業はメンテナンス中に実施するはずなので `--apply-immediately` オプションを付けて実行する。

```sh
db_identifier="xxxxxxxx"

aws rds modify-db-instance \
  --db-instance-identifier $db_identifier \
  --db-instance-class db.r5.12xlarge \
  --apply-immediately
```

RDS のイベントを確認し "Finished applying modification to DB instance class" というイベントメッセージを探す。

```sh
aws rds describe-events \
  --source-type db-instance \
  --source-identifier $db_identifier
```

時間がかかる変更の場合は定期的にウォッチすると良い。

```sh
while true; do
  aws rds describe-events \
    --source-type db-instance \
    --source-identifier $db_identifier;
  sleep 10;
done
```

## イベント通知を一時的に無効化する

RDS のイベント通知機能で failover などを通知している場合、メンテナンス作業によっても通知が飛んでしまう。これを回避するにはサブスクリプションを一時的に無効化すれば良い。

```sh
aws rds modify-event-subscription \
  --subscription-name xxxxxxxx \
  --no-enabled
```

作業が終わったら忘れずに有効化すること。

```sh
aws rds modify-event-subscription \
  --subscription-name xxxxxxxx \
  --enabled
```
