# ElastiCache for Redis

Terraform で ElastiCache for Redis (cluster mode disabled) を定義するには `aws_elasticache_cluster` と `aws_elasticache_replication_group` の 2 つのリソースがある。

基本的には `aws_elasticache_replication_group` を使えば良い。 Multi-AZ で可用性を高めるなら `number_cache_clusters` を 2 以上にして `automatic_failover_enabled` を true にする。

```terraform
resource "aws_elasticache_replication_group" "example" {
  replication_group_id          = "example"
  replication_group_description = "example"
  node_type                     = "cache.t3.micro"
  number_cache_clusters         = 2
  automatic_failover_enabled    = true
  engine_version                = "5.0.6"
  parameter_group_name          = "default.redis5.0"
}
```

staging など可用性を落としてコストを抑えたい場合は、ノード数を変数にして `automatic_failover_enabled` を切り替えると良い。

```terraform
variable "node_count" {
  type    = number
  default = 1
}

resource "aws_elasticache_replication_group" "example" {
  replication_group_id          = "example"
  replication_group_description = "example"
  node_type                     = "cache.t3.micro"
  number_cache_clusters         = var.node_count
  automatic_failover_enabled    = var.node_count == 1 ? false : true
  engine_version                = "5.0.6"
  parameter_group_name          = "default.redis5.0"
}
```

`automatic_failover_enabled` の値を後から変更する場合は `apply_immediately` を指定しないと次のメンテナンスウィンドウで適用される。それまでは意図しない差分が出てしまうので要注意。

## リードレプリカの追加

読み込み性能を上げるためにリードレプリカを追加する場合は `aws_elasticache_cluster` で明示的にノードを増やす。

```terraform
resource "aws_elasticache_replication_group" "example" {
  replication_group_id          = "example"
  replication_group_description = "example"
  node_type                     = "cache.t3.micro"
  number_cache_clusters         = 2
  automatic_failover_enabled    = true
  engine_version                = "5.0.6"
  parameter_group_name          = "default.redis5.0"

  lifecycle {
    ignore_changes = ["number_cache_clusters"]
  }
}

resource "aws_elasticache_cluster" "replica" {
  cluster_id           = "replica"
  replication_group_id = aws_elasticache_replication_group.example.id
}
```

このとき `aws_elasticache_replication_group` 側に `lifecycle` を指定しないと、ノードを削除したときに意図しないエラーが発生する。

> Error: error deleting Elasticache Cache Cluster (\<node name\>) (removing replica): CacheClusterNotFound: CacheCluster not found: \<node name\>
