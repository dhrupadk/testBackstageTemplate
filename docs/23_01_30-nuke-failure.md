# Postmortem - AWS-Nuke Did Not Execute (CDX-3887)
## Incident Date
01/30/23

## Authors
Fenil, Mitch

## Status
Complete

## Summary
The nuke service did not run over the weekend, no notifications were published, no resources destroyed

## Impact
All AWS Sandbox Accounts had resources that were not destroyed, resulting in extra cost to Toyota for resources running another week (roughly estimate 0-5k in cost)

## Detection
Team was monitoring, manually, in addition to datadog, having recently deployed a new version of nuke. Manually detected on Monday morning to due lack of logs in Datadog.

## Root Cause
Latest deployment missed dependent packages, causing failure of python nuke script execution. Furthermore, cron console logs were not being output to a file and thus could not be investigated.

## Resolution
- Resolve missing dependency issues
- Log cron job execution
  
### Resolution Steps

1. Missing python packages for boto3 and requests were installed
2. Dependencies were uploaded to the s3 bucket
3. Commands such as the below were executed to copy over missing dependent files from s3 to the nuke instance
    ```
    aws s3 cp  s3://<bucket>/cloud-nuke_linux_amd64 /home/ssm-user/aws-sandbox-nuke/prod/
    aws s3 cp  s3://<bucket>/s3_config.yaml /home/ssm-user/aws-sandbox-nuke/prod/
    ```
4. Crontab configuration was updated to output console into a file so that it could be ingested in datadog such as:
   ```
   0 18 * * 6 python3 /home/ssm-user/aws-sandbox-nuke/prod/sandbox-cloud-nuke-wrapper-forced.py /home/ssm-user/aws-sandbox-nuke/prod/aws-sandbox-nuke-cron-forced.log (new starting here) >> /home/ssm-user/aws-sandbox-nuke/prod/console_nuke_wrapper_forced.log 2>&1
   ```

## Timeline
- 1/30/23 - Issue found
- 1/31/23 - Initial investigation and issues found around missing dependencies
- 2/1/23 - Initial fixes deployed with missing packaged installed
- 2/1/23 - Repeat failure, more missing dependencies identified
- 2/2/23 - Redeployment with fixes, successful dryrun execution
- 2/8/23 - Successful execution confirmed
- 2/13/23 - Missing nuke alert action item completed

## Action Items
|Item|Type|Owner|Status|
|--|--|--|--|
|Redesign nuke to be serverless / more streamlined and codify triggers|prevent|SRE team|Ticket Created|
|Update cron commands to log console output and push to DD|mitigate|Fenil|Done|
|Install dependencies and dryrun|mitigate|Fenil|Done|
|New alert on no nuke execution in a week|mitigate|Fenil|Done|

## Lessons Learned
### What went well
- We were monitoring in advance having just deployed a change and caught it quick!
- Prompt in RCA and identifying resolution

### What went wrong
- Missed steps in deployment that were not planned/captured
- Not everything (instance userdata) was codified
- Missed potential hole in logs (crontab output)

## Supporting Material
[Monitor that did not fire](https://app.datadoghq.com/monitors/108635393?live=1w)

