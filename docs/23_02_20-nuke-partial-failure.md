# Postmortem - AWS-Nuke Partial Failure (CDX-4181)
## Incident Date
02/20/23

## Authors
Fenil

## Status
Complete

## Summary
The nuke service ran, however some error alerts fired

## Impact
Some accounts were nuked successfully while others were untouched. Based on investigation, 9 of 55 sandbox accounts were nuked successfully

## Detection
Datadog alerts triggered, notifying the team of fatal errors in the nuke process, spurring investigation and RCA.

## Root Cause
The nuke script(s) did not properly account for paginated responses, as a result the nuke process would run on an initial set of accounts in one OU, but never execute the last few of accounts in that OU and would not proceed to the second sandbox OU.

Furthermore, the number of log files exceeded the max default allowed by the data-dog agent to be live tailed and so the latest, post-fix, execution logs were not in Datadog.

## Resolution
- A code-fix addressing the error from a missing 'NextToken' value on the final paginated response from AWS' Organizations API
- Older log files, > 14 days, were pruned
  
### Resolution Steps

1. Identify affected code on sandbox-cloud-nuke-wrapper-forced.py
2. Rectify and test locally
3. Identify similar failure points in other nuke scripts and rectify + test
4. Commit and Review
5. Deploy

## Timeline
- 2/19/23 - Alerts fire
- 2/20/23 - Investigation begins, team begins looking at logs
- 2/20/23 - Issue identified, testing of fix in progress
- 2/20/23 - Fix completed, pushed for review
- 2/22/23 - Review approved and fix is deployed
- 2/23/23 - New iteration of nuke notifications runs successfully
- 2/23/23 - Logs identified as missing in Datadog
- 2/23/23 - Investigation into RCA reveals the max count of tail-able logs (500) exceeded
- 2/23/23 - Older log files pruned
- 2/23/23 - Logs confirmed ingesting successfully
  
## Action Items
|Item|Type|Owner|Status|
|--|--|--|--|
|Add a recurring job on the server to prune older log files | Prevent | Fenil | Story Captured |
|Create key metrics/SLXs for nuke in DD beyond failure scenarios | Prevent | Fenil | Story Captured |

## Lessons Learned
### What went well
- Quick response and resolution

### What went wrong
- This error has been log-present and was never caught
- No KPI around number of accounts nuked / notified to have seen this earlier

## Supporting Material
[Event triggering alert](https://app.datadoghq.com/event/explorer?cols=&event=AQAAAYZltZNg_RlSugAAAABBWVpsdFpOZ0FBQ0FYbE15X05PWFhEcEc&messageDisplay=expanded-lg&options=&sort=DESC&from_ts=1677168839947&to_ts=1677169739947&live=true)
