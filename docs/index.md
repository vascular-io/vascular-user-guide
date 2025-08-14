# Welcome to Vascular Platform

Thank you for choosing Vascular. This guide will walk you through registration, setup, and integration.

---

## 1. Register

To get started, please [register an account](https://yourdomain.com/register).

---

## 2. Sign In

If you already have an account, [sign in here](https://yourdomain.com/signin).

---

## 3. Running Vascular

Before running Vascular, ensure you meet the following requirements.

### 3.1 Prerequisites

Before you begin, make sure you have:

- An active subscription  
- Access credentials for our private container registry  
  *(Vascular is distributed as a container image.)*  
- A valid license file (`license.key`)  
  You can download this from your Customer Dashboard after registration and payment

---

### 3.2 Pull the Image from the Registry

Use the following command to pull the latest version of the Vascular container:

```bash
docker pull vascular.registry.com/inbox:latest
```


### 3.3 Activating the Application

The application requires the license file to start.
You can provide the license file in two ways:

#### 1. Default location
Mount the license file inside the container at:


```bash
/etc/vascular-inbox/license.key
```

#### 2. Custom location
Mount the license file anywhere you prefer and provide the path using:

```bash
--license-file=/custom/path/license.key
```

If the license file is missing or invalid, the application will not start.

**Example: Kubernetes Deployment with Custom License File Path**

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vascular-inbox
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vascular-inbox
  template:
    metadata:
      labels:
        app: vascular-inbox
    spec:
      containers:
        - name: vascular-inbox
          image: vascular.registry.com/inbox:latest
          args: ["--license-file=/etc/vascular-inbox/license.key"] # optional if using default
          volumeMounts:
            - name: license
              mountPath: /etc/vascular-inbox/license.key
              subPath: license.key
              readOnly: true
      volumes:
        - name: license
          secret:
            secretName: vascular-license

```

## 4. Customizing Application Features

Vascular supports runtime customization through environment variables. These variables control optional features and integrations without requiring code changes.

Set these environment variables using:

- `docker run -e VAR_NAME=value`
- In your `docker-compose.yml` or Kubernetes `env:` block
- As part of your deployment configuration


### 4.1 Available Environment Variables

| Variable Name     | Default | Type    | Description                                                                 |
|-------------------|---------|---------|-----------------------------------------------------------------------------|
| `SERVER_PORT`     | `3000`                            | String  | Port the application listens on.|
| `DATABASE_URL`    | _(required)_                      | String  | PostgreSQL connection string. Must include protocol, user, password, host, port, and db name. |
| `LICENSE_KEY`     | `/etc/vascular-inbox/license.key` | String  | Full path to the license file used to activate the application. Must be mounted at runtime.   |
| `TENANT_ID`       | _(required)_                      | String  | Microsoft Entra (Azure AD) tenant ID used for validating OAuth tokens. |
| `AWS_REGION`      | `eu-west-1`                       | String  | AWS region where Kinesis Firehose is hosted. Used to stream analytics event records to Firehose. |
---


## 5. Client Integration

You can integrate Vascular with your client applications using one of our open-source plugins.

### 5.1 Download Client Plugins

Web (React)
https://github.com/vascular-io/vascular-js

React Native
https://github.com/vascular-io/vascular-js

Android (Java/Kotlin)
https://github.com/vascular-io/vascular-kotlin

iOS (Swift)
https://github.com/vascular-io/vascular-swift

Flutter
https://github.com/vascular-io/vascular-flutter


## 6. Streaming Analytics Events to S3

The Vascular platform supports streaming of analytics events to **Amazon S3** using **Amazon Kinesis Data Firehose**. This allows persistent storage of user event logs for auditing, analytics, and integration with analyticals tools.

Four types of events are tracked and streamed:

- Delivered
- Read
- Opened
- Deleted

Each event type is streamed to its own dedicated S3 bucket.

---

### 6.1 Required S3 Buckets

You must create the following four S3 buckets:

| Event Type | Bucket Name (example) |
|------------|------------------------|
| Delivered  | `delivered`            |
| Read       | `read`                 |
| Opened     | `opened`               |
| Deleted    | `deleted`              |

> Ensure each bucket has the correct write permissions and optional lifecycle policies (e.g., auto-expiration) as needed.

---

### 6.2 IAM Role for Firehose

Create a single IAM role named:

```plaintext
firehose-to-s3-writer
```

This role grants Kinesis Firehose permission to write to the S3 buckets.

**Trust Relationship Policy**
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "firehose.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

```
**Permissions Policy**
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:PutObjectAcl"],
      "Resource": [
        "arn:aws:s3:::delivered/*",
        "arn:aws:s3:::read/*",
        "arn:aws:s3:::opened/*",
        "arn:aws:s3:::deleted/*"
      ]
    }
  ]
}

```

### 6.3 Create Firehose Delivery Streams

Create four Firehose delivery streams, one for each event type:

- **delivered-s3-stream** → writes to the `delivered` bucket
- **read-s3-stream** → writes to the `read` bucket
- **opened-s3-stream** → writes to the `opened` bucket
- **deleted-s3-stream** → writes to the `deleted` bucket

Use the IAM role `firehose-to-s3-writer` for all streams to grant write permissions to the target S3 buckets.

### 6.4 Example: deleted-s3-stream Configuration

The following is a sample JSON configuration for the `deleted-s3-stream`. It defines how Firehose writes analytics events to the `deleted` S3 bucket:

```json
{
  "DeliveryStreamName": "deleted-s3-stream",
  "DeliveryStreamType": "DirectPut",
  "S3DestinationConfiguration": {
    "RoleARN": "arn:aws:iam::000000000000:role/firehose-to-s3-writer",
    "BucketARN": "arn:aws:s3:::deleted",
    "Prefix": "myorg-log",
    "ErrorOutputPrefix": "error-log/",
    "BufferingHints": {
      "SizeInMBs": 1,
      "IntervalInSeconds": 60
    },
    "CompressionFormat": "UNCOMPRESSED",
    "CloudWatchLoggingOptions": {
      "Enabled": false,
      "LogGroupName": "",
      "LogStreamName": ""
    }
  },
  "Tags": [
    {
      "Key": "Environment",
      "Value": "Production"
    }
  ]
}
```

### 6.5 Notes

- **S3 Prefix**: Files will be written to S3 under the prefix `myorg-log/YYYY/MM/DD/HH/`, based on the time the data was delivered.
- **Buffering**: Records are flushed to S3 when either 1 MB of data is accumulated or 60 seconds have passed.
- **Compression**: By default, data is stored uncompressed. You can change the compression format to `GZIP` or `Snappy` if needed.
- **Error Output Prefix**: If delivery to S3 fails, failed records are written to the `error-log/` prefix for inspection.
- **CloudWatch Logging**: Logging is disabled in the example config. Enable it by setting `Enabled: true` and providing a log group and stream if needed.
- **Region Alignment**:
  - Make sure the environment variable `AWS_REGION` is set in your application.
  - The **S3 buckets** and **Firehose delivery streams** should be created in the **same AWS region** to avoid latency, cross-region data transfer costs, and potential access issues.
```

