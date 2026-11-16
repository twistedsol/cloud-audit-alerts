# Cloud Audit Alerts (AWS)

Get an email within about a minute when someone makes a risky S3 change in an AWS account.

Part of my 30-day cloud-security challenge: **Scan it. Detect it. Harden it.** This is the "detect" part, built by hand in the AWS console on a free-plan account.

## What it does

When any of these S3 actions happens in the account, an email arrives with the full event details:

- `CreateBucket` (a new bucket appears)
- `PutBucketPolicy` (a bucket policy is changed)
- `DeletePublicAccessBlock` (a bucket's public-access protection is removed)

## How it works

```
S3 action  ->  CloudTrail  ->  EventBridge rule  ->  SNS topic  ->  email
```

| Piece | Name | Job |
|-------|------|-----|
| CloudTrail trail | `management-events` | Records management activity in the account |
| EventBridge rule | `s3-risky-changes` | Matches the risky S3 events on the default bus |
| SNS topic | `security-alerts` | Sends the matched event to my confirmed email address |

Region: `eu-north-1` (Stockholm). The rule and the SNS topic must be in the same region as the events they watch.

## The event pattern

```json
{
  "source": ["aws.s3"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["s3.amazonaws.com"],
    "eventName": ["CreateBucket", "PutBucketPolicy", "DeletePublicAccessBlock"]
  }
}
```

## How I tested it

1. Created a test bucket in S3.
2. The alert email arrived in about a minute and showed `"eventName":"CreateBucket"` with the bucket name, the time, the AWS region, which role did it, and the source IP.
3. Deleted the test buckets afterwards.

The first test bucket's email was missed, so I checked each link in the chain instead of guessing:
- The rule's **Monitoring** tab showed 1 matched event, 1 invocation and no failed invocations, so the rule worked.
- A manual **Publish message** test in SNS arrived in my inbox, so email delivery worked.
- A second test bucket then produced the alert email.

## What the alert tells you

The email is the raw CloudTrail event. The fields worth reading first:

- `eventName`: what happened
- `userIdentity`: who did it (here, an assumed role)
- `eventTime` and `awsRegion`: when and where
- `sourceIPAddress`: where the request came from
- `requestParameters`: what it was done to (the bucket name)

**Note:** real alerts contain the AWS account ID, IP address and key IDs. Redact them before sharing a screenshot.

## What I learned

- A detection pipeline is a chain, and when something is silent the way to find the broken link is to test each one in turn.
- The EventBridge Monitoring graphs default to a 1-week window, so an older event looks like "No data" until the date range is widened.
- The newer EventBridge console has an "Enhanced builder" that is drag-and-drop. The "Advanced builder" is easier for a first rule because every setting is visible.
- A CloudTrail trail needs an S3 bucket to store its logs, which is the one small cost in this setup.
- This alert shows who did something and from where, which is the information an investigation starts with.

## Limits and next ideas

- Only three S3 actions are watched. Next rule: IAM changes, which need a different region for the global IAM events.
- It is email only. Next: send alerts to a chat channel or a ticket.
- Filter out expected activity so the alerts stay worth reading.
- Manual console setup. Next: rebuild it in Terraform and scan it with Checkov, like my earlier labs.
