# CSV to Redshift Spectrum
2020-02-20

Initial desired result: Python service -> CSV -> S3 -> Redshift (Spectrum)

Written for my own sake, may have errors or cases where it doesn't work. Not used for crucial data.

## Dump to CSV.
```
with open(local_location, 'w') as csvfile:
    fieldnames = data[0].keys()
    writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(data)
```

## Upload file
```
split = filename.split('T') # 2020-02-19T08:54:13.443244.parquet
key = s3_prefix + split[0] + '/' + split[1] # partitioned on date
upload_file(local_location, s3_bucket, key)
```

See [upload_file](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/s3-uploading-files.html).

## Glue crawler
Crawl the `s3_prefix` location above with a Glue crawler, change the partition column's name to `date` or similar (select table, edit schema).

## Add table to Redshift
```
create external schema <redshift schema> from data catalog
database '<Glue database>'
iam_role '<ARN>'
region 'eu-west-1';
```

Gotcha: It's not enough that a new role can be assumed by Redshift, you need to explicitly add it to the cluster.

<details>
    <summary>IAM permissions needed</summary>

    Minimum required according to Amazon, see https://docs.aws.amazon.com/redshift/latest/dg/c-spectrum-iam-policies.html.

    ```
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:GetBucketLocation",
                    "s3:GetObject",
                    "s3:ListMultipartUploadParts",
                    "s3:ListBucket",
                    "s3:ListBucketMultipartUploads"
                ],
                "Resource": [
                    "arn:aws:s3:::bucketname",
                    "arn:aws:s3:::bucketname/folder1/folder2/*"
                ]
            },
            {
                "Effect": "Allow",
                "Action": [
                    "glue:CreateDatabase",
                    "glue:DeleteDatabase",
                    "glue:GetDatabase",
                    "glue:GetDatabases",
                    "glue:UpdateDatabase",
                    "glue:CreateTable",
                    "glue:DeleteTable",
                    "glue:BatchDeleteTable",
                    "glue:UpdateTable",
                    "glue:GetTable",
                    "glue:GetTables",
                    "glue:BatchCreatePartition",
                    "glue:CreatePartition",
                    "glue:DeletePartition",
                    "glue:BatchDeletePartition",
                    "glue:UpdatePartition",
                    "glue:GetPartition",
                    "glue:GetPartitions",
                    "glue:BatchGetPartition"
                ],
                "Resource": [
                    "*"
                ]
            }
        ]
    }
    ```
</details>

## CSV formatting
Realize your CSV has line breaks and that Spectrum doesn't seem to handle that no matter how much you try to escape and quote your data.

## Enter Parquet
Replace CSV dumping with Parquet dumping, because Parquet doesn't care about your line breaks.

```
import numpy as np
import pyarrow as pa
import pandas as pd
import pyarrow.parquet as pq

df = pd.DataFrame(data)
table = pa.Table.from_pandas(df)
pq.write_table(table, local_location, use_deprecated_int96_timestamps=True)
```

# Outcome
Python service -> Parquet -> S3 -> Redshift (Spectrum)

Replacing line breaks probably would have worked fine, but replacing CSV with a columnar data store so far has no real downsides.
