### AWS - cloudwatch alert

------

###### use case: massive memory server (ec2) switched on for 24 hours

###### action: create a cloudwatch alert

###### services: cloudwatch, SNS, IAM, EC2

------

#### role and policy

— Create a policy to allow ec2 instance to post custom metrics and name it `allow-post-metrics`(Policy Generator: https://awspolicygen.s3.amazonaws.com/policygen.html) 

— Attach policy to a role eg. Named `cloudwatch-alert`

Sample Policy using policy generator: 

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1518047916513",
      "Action": [
        "cloudwatch:PutMetricData",
        "cloudwatch:GetMetricData"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

#### create cloudwatch metric

###### for help, use: eg. `aws cloudwatch put-metric-data help` for parameters

Type into the command line: 

```
aws cloudwatch put-metric-data \
  --namespace Put-Custom-Metric-Data \
  --metric-name Massive-Memory-Alive \
  --value 1 \
  --region ap-southeast-2
```

This will tell AWS that you have just created a custom metric.

We now have to create a file in ```/opt``` so that when we create our cronjob - it knows where to look to run the script. 

/opt/post_cloudwatch looks as so:

```!/bin/bash
#!/bin/bash

aws cloudwatch put-metric-data \
  --namespace rebecca \
  --metric-name foobar \
  --value 1 --region ap-southeast-2
```

-   Note /opt/ directory 

#### install cronjob

type `crontab -e` to edit your system's cronjob. Comment what the cronjob is doing

File looks as so:

```
# cronjob to post metrics
* * * * * /opt/post_cw
```

#### cloudwatch

— Go to **Alarm > Create alarm > Custom metric > 1. Select metric 2. Define alarm**

— Input **name** and **description**

Whenever: `{metric name}` is >= 1 (which is our value we set in our metric - a heartbeat when the instance is on)

for **24 out of 24 datapoints**, the alarm is go off when this condition is met. 

###### Alarm Preview

— period: 1 hour

— statistics: sum

The **alarm preview** **period** means every 1 hour, cloudwatch wakes up and checks for a datapoint. Once the sum of all the data points add up to 24 data points, the alarm will be triggered. 

###### Additional Settings

— Treat missing data as: `good (not breaching threshold)`

###### Action

Whenever this alarm: `state is Alarm`

Send notification to: `{Topic}`

#### **SNS simple notification service**

— **Create Topic > Input topic name & Display name > Subscribe to topic** 

