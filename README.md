# How to migrate AWS ElasticSearch from 6.0 to 6.2

#### 1 - Create a S3 bucket

#### 2 - Create a new ElasticSearch domain

#### 3 - Create IAM Role + Policy

```
variable "elasticsearch_manual_snapshots_bucket_name" {
  default = "bucket-name-here"
}

resource "aws_iam_role" "elasticsearch_default_iam_role" {
  name        = "DefaultRoleForElasticSearch"
  description = "Managed by Terraform"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Principal": {
        "Service": "es.amazonaws.com"
      },
      "Effect": "Allow"
    }
  ]
}
EOF
}

resource "aws_iam_policy" "elasticsearch_default_iam_policy" {
  name = "DefaultPolicyForElasticSearch"

  policy = <<EOF
{
   "Version":"2012-10-17",
   "Statement":[
       {
           "Action":[
               "s3:ListBucket"
           ],
           "Effect":"Allow",
           "Resource":[
               "arn:aws:s3:::${var.elasticsearch_manual_snapshots_bucket_name}"
           ]
       },
       {
           "Action":[
               "s3:GetObject",
               "s3:PutObject",
               "s3:DeleteObject"
           ],
           "Effect":"Allow",
           "Resource":[
               "arn:aws:s3:::${var.elasticsearch_manual_snapshots_bucket_name}/*"
           ]
       }
   ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "elasticsearch_role" {
  role       = "${aws_iam_role.elasticsearch_default_iam_role.name}"
  policy_arn = "${aws_iam_policy.elasticsearch_default_iam_policy.arn}"
}
```

#### 4 - Setup script to create snapshot repos on ES 

Python deps

```
pip install requests requests-aws4auth 
```

Edit AWS_*, region, host, bucket, role_arn contents and save it as something.py
```
# sourced from: https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-managedomains-snapshots.html#es-managedomains-snapshot-registerdirectory

import requests
import sys

from requests_aws4auth import AWS4Auth

AWS_ACCESS_KEY_ID='########'
AWS_SECRET_ACCESS_KEY='########'
region = '####REGION####'
service = 'es'

awsauth = AWS4Auth(AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, region, service)

host = sys.argv[1]

# REGISTER REPOSITORY

path = '_snapshot/manual-snapshots-repo' # the Elasticsearch API endpoint
url = host + path

payload = {
  "type": "s3",
  "settings": {
    "bucket": "####BUCKET_NAME####",
    "region": "####REGION####",
    "role_arn": "arn:aws:iam::####ACCOUNT_ID####:role/DefaultRoleForElasticSearch"
  }
}

headers = {"Content-Type": "application/json"}

r = requests.put(url, auth=awsauth, json=payload, headers=headers) # requests.get, post, put, and delete all have similar syntax

print(r.text)
```

#### 5 - Register repos against two elasticsearch domains (include https:// and trailing /)

```
python something.py https://####OLD_ES_ENDPOINT###.eu-west-1.es.amazonaws.com/
```

```
python something.py https://####NEW_ES_ENDPOINT###.eu-west-1.es.amazonaws.com/
```
#### 6 - Create snapshot

```
# curl -XPUT 'https://####OLD_ES_ENDPOINT###.eu-west-1.es.amazonaws.com/_snapshot/manual-snapshots-repo/60-snapshot-for-migration'
{"accepted":true}
```

#### 7 - Check if snapshot creation is finished

```
# curl -XGET 'https://####OLD_ES_ENDPOINT###.eu-west-1.es.amazonaws.com/_snapshot/manual-snapshots-repo/_all?pretty'
```

STATE should be "SUCCESS"

### 8 - Restore snapshot in the new domain

Delete all indices first
```
# curl -XDELETE 'https://####NEW_ES_ENDPOINT###.eu-west-1.es.amazonaws.com/_all'
{"acknowledged":true}
```

Restore
```
# curl -XPOST 'https://####NEW_ES_ENDPOINT###.es.amazonaws.com/_snapshot/manual-snapshots-repo/60-snapshot-for-migration/_restore'
{"accepted":true}
```
