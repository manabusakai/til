# CloudWatch Metric Alarm

CloudWatch のメトリクスをトリガーに Auto Scaling でスケーリングするには `aws_cloudwatch_metric_alarm` と `aws_autoscaling_policy` を定義する。

次のような Auto Scaling Group がある。

```terraform
resource "aws_launch_template" "web" {
  name          = "web"
  image_id      = "ami-052652af12b58691f"
  instance_type = "t3.nano"
}

resource "aws_autoscaling_group" "web" {
  name               = "web"
  min_size           = 0
  max_size           = 1
  availability_zones = ["ap-northeast-1a", "ap-northeast-1c"]

  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }
}
```

この Auto Scaling Group に対して「CPU 使用率が 2 分間 70% 以上になったら 1 台増やす」というポリシーを定義する。

```terraform
resource "aws_autoscaling_policy" "add_capacity" {
  autoscaling_group_name = aws_autoscaling_group.web.name
  name                   = "Add capacity"
  policy_type            = "SimpleScaling"
  adjustment_type        = "ChangeInCapacity"
  scaling_adjustment     = 1
}

resource "aws_cloudwatch_metric_alarm" "web_cpu_utilization" {
  alarm_name          = "Web CPU Utilization"
  namespace           = "AWS/EC2"
  metric_name         = "CPUUtilization"
  threshold           = "70"
  period              = "60"
  evaluation_periods  = "2"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  statistic           = "Average"
  alarm_actions       = [aws_autoscaling_policy.add_capacity.arn]

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.web.name
  }
}
```

Auto Scaling Group にポリシーを付与するには次の権限が必要である。

* `autoscaling:Describe*`
* `autoscaling:*Policy`

ref. https://docs.aws.amazon.com/autoscaling/ec2/APIReference/API_Operations.html
