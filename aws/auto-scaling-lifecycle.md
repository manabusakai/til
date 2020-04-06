# Auto Scaling Lifecycle

Auto Scaling Group で管理された特定のインスタンスに対してよく行う操作。

## スタンバイ状態にする

スタンバイ状態に移行させることで一時的に Auto Scaling Group のヘルスチェックから外せる。 ELB に紐づく場合は ELB からも切り離される。インスタンスを再起動させたり、調査のために一時的にサービスから切り離すときに使う。

```sh
aws autoscaling enter-standby \
  --instance-ids i-xxxxxxxxxxxxxxxxx \
  --auto-scaling-group-name asg-name \
  --should-decrement-desired-capacity
```

スタンバイ状態を解除する。

```sh
aws autoscaling exit-standby \
  --instance-ids i-xxxxxxxxxxxxxxxxx \
  --auto-scaling-group-name asg-name
```

## Auto Scaling Group から特定のインスタンスを終了する

Auto Scaling Group から特定のインスタンスを終了するには次のコマンドを使う。

```sh
aws autoscaling terminate-instance-in-auto-scaling-group \
  --instance-id i-xxxxxxxxxxxxxxxxx \
  --should-decrement-desired-capacity
```

`aws ec2 terminate-instances` を使うと Auto Scaling Group のライフサイクルを通らないため安全に終了しない。たとえば、Lifecycle hook や ELB の Connection draining など。
