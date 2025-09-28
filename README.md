Assignment 2.14

Answer:

1) 	Does SNS guarantee exactly-once delivery to subscribers?

No. SNS does not ensure delivery to subscribers happens exactly once. 

For SNS (Standard topics), the delivery occurs at least once with an effort to maintain order. Subscribers may encounter repeated and unordered messages. It is essential to design consumers to be idempotent.

For SNS FIFO topics, it offers ordered and deduplicated delivery to compatible FIFO targets, typically SQS FIFO. By linking SNS FIFO to SQS FIFO and implementing deduplication (using content-based methods or a MessageDeduplicationId), you can effectively achieve exactly-once delivery to the queue. However, achieving end-to-end "exactly once" at the application level relies on your processing being idempotent and ensuring that your storage or side effects do not repeat tasks during retries.

In conclusion, the guideline to remember:  
Standard SNS → consider it as at least once.  
SNS FIFO + SQS FIFO → you can maintain strict order and deduplication in the queue, but it’s crucial to ensure idempotency in your handlers.


2) 	What is the purpose of the Dead-letter Queue (DLQ)? This is a feature available to - SQS/SNS/EventBridge.

A DLQ serves as a backup system (an SQS queue) that collects messages or events that failed to be processed or delivered after the service has used its retry options. It stops harmful messages from trying again indefinitely and allows for examination, prioritization, and reprocessing at a later time.

How different services utilize a DLQ:

• SQS → DLQ (through Redrive Policy):
When a message is received more than the maxReceiveCount without deletion (for instance, if your consumer fails), SQS transfers it to the linked DLQ.

• SNS subscription → DLQ:
If SNS cannot successfully send a published message to a subscription endpoint after multiple attempts (like an SQS queue, Lambda, or HTTP endpoint), it can transfer that message to the subscription's DLQ (which is another SQS queue) if one has been set up.

• EventBridge rule → DLQ:
If EventBridge is unable to trigger the target (or if the event transformation or target errors continue after retries), it can relay the original event to the DLQ associated with the rule.

Reasons DLQs are significant:
• Prevent infinite retries ("poison pill" protection)
• Keep unsuccessful messages for analysis
• Allow for organized redrive or replay once the issue or downstream problem is resolved
• Provide clear indicators for monitoring (for example, "DLQ has more than 0 messages")

3)	How would you enable a notification to your email when messages are added to the DLQ?

Since DLQs function as SQS queues, the most straightforward and dependable method is to set a CloudWatch alarm on the SQS metrics of the DLQ that sends notifications to an SNS topic, where you have your email linked.

A)	Quick console instructions (Suitable for SQS/SNS/EventBridge DLQs)

1. 	Establish an SNS topic (for example, DLQAlerts) in SNS.
- Subscribe your email address (make sure to confirm the received email).

2. 	Set a CloudWatch alarm on your DLQ queue:
- Metric:	AWS/SQS → Queue Metrics → ApproximateNumberOfMessagesVisible
- Dimension: 	QueueName = <your DLQ name>
- Threshold: 	Static > 0 (1 data point is adequate for timely notifications)
- Period: 	1 minute (or 5 if you want fewer alerts)
- Alarm action: 	Select the DLQAlerts SNS topic from step 1.

3. 	(Optional) Include a second alarm for AgeOfOldestMessage > 0 (or > N seconds) to identify stalled processing.

4. 	(Optional) If you have multiple DLQs, implement a composite alarm that activates when any individual queue alarm is triggered.

This method is effective regardless of whether the DLQ is linked to SQS, an SNS subscription, or an EventBridge 
rule since in all scenarios, the DLQ operates as an SQS queue with identical metrics.

B) 	Terraform example (parameterized)

This snippet creates:
•	An SNS topic for alerts + an email subscription.
•	A CloudWatch alarm that fires when any messages are visible in each DLQ you list.

variable "dlq_queue_names" {
  type    = list(string)
  default = ["orders-dlq", "payments-dlq"] # ← replace with your DLQ queue names
}

variable "alert_email" {
  type    = string
  default = "you@example.com"              # ← replace with your email
}

resource "aws_sns_topic" "dlq_alerts" {
  name = "DLQAlerts"
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.dlq_alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email
}

# One alarm per DLQ for "messages visible > 0"
resource "aws_cloudwatch_metric_alarm" "dlq_has_messages" {
  for_each            = toset(var.dlq_queue_names)
  alarm_name          = "DLQ-${each.key}-messages-visible"
  alarm_description   = "Alert when ${each.key} has any visible messages."
  namespace           = "AWS/SQS"
  metric_name         = "ApproximateNumberOfMessagesVisible"
  statistic           = "Maximum"
  period              = 60
  evaluation_periods  = 1
  comparison_operator = "GreaterThanThreshold"
  threshold           = 0
  treat_missing_data  = "notBreaching"

  dimensions = {
    QueueName = each.key
  }

  alarm_actions = [aws_sns_topic.dlq_alerts.arn]
  ok_actions    = [aws_sns_topic.dlq_alerts.arn]
}

# Optional: alert if the oldest message is getting stale (e.g., > 300s = 5 minutes)
resource "aws_cloudwatch_metric_alarm" "dlq_age_stale" {
  for_each            = toset(var.dlq_queue_names)
  alarm_name          = "DLQ-${each.key}-oldest-msg-age"
  alarm_description   = "Alert when ${each.key}'s oldest message is older than 5 minutes."
  namespace           = "AWS/SQS"
  metric_name         = "ApproximateAgeOfOldestMessage"
  statistic           = "Maximum"
  period              = 60
  evaluation_periods  = 1
  comparison_operator = "GreaterThanThreshold"
  threshold           = 300
  treat_missing_data  = "notBreaching"

  dimensions = {
    QueueName = each.key
  }

  alarm_actions = [aws_sns_topic.dlq_alerts.arn]
  ok_actions    = [aws_sns_topic.dlq_alerts.arn]
}



Notes
-	You need to confirm your email for the SNS subscription before you can get alerts.
-	For DLQs in SNS subscriptions and EventBridge rules, keep directing the 
alarm to the name of the DLQ queue—the source of the metrics is always the SQS queue.
-	If you desire one alarm for "any DLQ has messages," set up a composite alarm 
that combines the individual queue alarms mentioned above.
-	Maintain idempotency in consumers (utilize natural keys, conditional writes, or de-
duplication tables) so that retries or occasional duplicates do not create double-effects.
