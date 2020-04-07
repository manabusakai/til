# Change the routing policy for Route 53 records

Terraform を使って Route 53 レコードのルーティングポリシーを変更するには、いくつかのステップを踏む必要がある。ここでは、ダウンタイムなしにルーティングポリシーを Simple から Weighted に変更する手順を検証する。

`test.manabusakai.com` のレコードを `1.1.1.1` と `2.2.2.2` を 50 % ずつ返す Weighted ルーティングポリシーに変更する。

```terraform
resource "aws_route53_record" "test" {
  zone_id = "Z11GV4TXXXXXXX"
  name    = "test.manabusakai.com"
  type    = "A"
  ttl     = "10"
  records = ["1.1.1.1"]
}
```

まずはリソース名を分かりやすくするために `terraform state mv` する。今回は `test_blue` と `test_green` というリソース名にする。

```
$ terraform state mv aws_route53_record.test aws_route53_record.test_blue
Move "aws_route53_record.test" to "aws_route53_record.test_blue"
Successfully moved 1 object(s).
```

tfstate が変更されたのでそれに合うようにリソース名 (`test` → `test_blue`) を変更し、Weighted に必要な `set_identifier` と `weighted_routing_policy` を追加する。ここでいったん apply する。

```terraform
resource "aws_route53_record" "test_blue" {
  zone_id        = "Z11GV4TXXXXXXX"
  name           = "test.manabusakai.com"
  type           = "A"
  ttl            = "10"
  records        = ["1.1.1.1"]
  set_identifier = "blue"

  weighted_routing_policy {
    weight = 100
  }
}
```

次に `test_green` を定義する。 apply の順序によってはダウンタイムが発生する可能性があるので、`weight` は 100 対 0 にしておき `test_green` 側にはリクエストを流さないようにする。ここでも忘れずに apply する。

```terraform
resource "aws_route53_record" "test_blue" {
  zone_id        = "Z11GV4TXXXXXXX"
  name           = "test.manabusakai.com"
  type           = "A"
  ttl            = "10"
  records        = ["1.1.1.1"]
  set_identifier = "blue"

  weighted_routing_policy {
    weight = 100
  }
}

resource "aws_route53_record" "test_green" {
  zone_id        = "Z11GV4TXXXXXXX"
  name           = "test.manabusakai.com"
  type           = "A"
  ttl            = "10"
  records        = ["2.2.2.2"]
  set_identifier = "green"

  weighted_routing_policy {
    weight = 0
  }
}
```

最後に `weight` を任意の割合に変えれば良い。 `weight` の変更は update in-place で行われる。それぞれ 50 % ずつ返したいならこんな感じ。

```terraform
resource "aws_route53_record" "test_blue" {
  zone_id        = "Z11GV4TXXXXXXX"
  name           = "test.manabusakai.com"
  type           = "A"
  ttl            = "10"
  records        = ["1.1.1.1"]
  set_identifier = "blue"

  weighted_routing_policy {
    weight = 50
  }
}

resource "aws_route53_record" "test_green" {
  zone_id        = "Z11GV4TXXXXXXX"
  name           = "test.manabusakai.com"
  type           = "A"
  ttl            = "10"
  records        = ["2.2.2.2"]
  set_identifier = "green"

  weighted_routing_policy {
    weight = 50
  }
}
```

端折ってまとめて apply すると予期せぬエラーが発生するので、ステップごとに確認しながら進めるのがポイント。
