# AWS Athena

Scripts for setting up AWS Athena with common AWS Services.  Used in [this blog post](https://medium.com/@robwitoff/athena-alb-log-analysis-b874d0958909).

### ALBs

**Sample Request**
```bash
type string                       http
timestamp string,                 2016-12-04T04:01:17.332214Z
elb string,                       app/web/xxxxxxxxx
client string,                    123.45.6.7:50814
target string,                    -
request_processing_time int       -1
target_processing_time int        -1
response_processing_time int      -1
elb_status_code int               503
target_status_code string         -
received_bytes int                106
sent_bytes int                    386
request string                    "GET http://web-00000000.us-east-1.elb.amazonaws.com:80/ HTTP/1.1"
user_agent string                 "curl/7.49.0"
ssl_cipher string                 -
ssl_protocol string               -
target_group_arn string           arn:aws:elasticloadbalancing:us-east-1:0000000:targetgroup/web/00000000
trace_id string                   "Root=1-0000-0000ae"
```

**Parsing Regex**
```bash
type string,                      ([^ ]*)
timestamp string,                 ([^ ]*)
elb string,                       ([^ ]*)
client_ip string,                 ([^ ]*):([0-9]*)
client_port string,
target string,                    ([^ ]*)
request_processing_time int,      ([-0-9]*)
target_processing_time int,       ([-0-9]*)
response_processing_time int,     ([-0-9]*)
elb_status_code int,              ([-0-9]*)
target_status_code string,        ([^ ]*)
received_bytes int,               ([-0-9]*)
sent_bytes int,                   ([-0-9]*)
request_verb string,              \"([^ ]*) ([^ ]*) ([^ ]*)\"
request_url string,
request_proto string
user_agent string,                 \"([^\"]*)\"
ssl_cipher string,                 ([^ ]*)
ssl_protocol string,               ([^ ]*)
target_group_arn string,           ([^ ]*)
trace_id string                    ([^ ]*)
```

**DDL**
```bash
CREATE EXTERNAL TABLE IF NOT EXISTS logs.web_alb (
  type string,
  time string,
  elb string,
  client_ip string,
  client_port string,
  target string,
  request_processing_time int,
  target_processing_time int,
  response_processing_time int,
  elb_status_code int,
  target_status_code string,
  received_bytes int,
  sent_bytes int,
  request_verb string,
  request_url string,
  request_proto string,
  user_agent string,
  ssl_cipher string,
  ssl_protocol string,
  target_group_arn string,
  trace_id string
)
PARTITIONED BY(year string, month string, day string)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
WITH SERDEPROPERTIES (
  'serialization.format' = '1',
  'input.regex' = '([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*):([0-9]*) ([^ ]*) ([-0-9]*) ([-0-9]*) ([-0-9]*) ([-0-9]*) ([^ ]*) ([-0-9]*) ([-0-9]*) \"([^ ]*) ([^ ]*) ([^ ]*)\" \"([^\"]*)\" ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*)'
) LOCATION 's3://{{BUCKET_NAME}}/AWSLogs/{{ACCOUNT_ID}}/elasticloadbalancing/us-east-1/';

```

**Query**
```sql
/* Load Data Into Athena */
ALTER TABLE logs.web_alb add partition (year="2016", month="12", day="*")
location "s3://{{BUCKET_NAME}}/AWSLogs/{{ACCOUNT_ID}}/elasticloadbalancing/us-east-1/2016/12/";

/* Query */
SELECT * FROM logs.web_alb LIMIT 100;

/* 500s */
SELECT
  time,
  client_ip,
  elb_status_code,
  request_url,
  user_agent
FROM logs.web_alb
WHERE elb_status_code>=500 AND elb_status_code<600
ORDER BY time DESC
LIMIT 100;

/* Unique IPs */
SELECT client_ip, COUNT(*) as count
FROM logs.web_alb
WHERE elb_status_code>=500 AND elb_status_code<600
GROUP BY client_ip;

/* Unique IP + Paths */
SELECT client_ip, request_url, COUNT(*) as count
FROM logs.web_alb
WHERE elb_status_code>=500 AND elb_status_code<600
GROUP BY client_ip, request_url;
```

**Hide Results**
```javascript
$('#resultTable td:nth-child(3)').text('123.4.5.6')
$('#resultTable td:nth-child(5)').text('http://web.us-east-1.elb.amazonaws.com:80/login')
```
