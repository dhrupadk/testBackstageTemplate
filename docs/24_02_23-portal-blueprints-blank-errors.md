# Postmortem - Errors on creating blueprints

## Incident Date
02/24/23

## Authors
Fenil

## Status
Complete

## Summary
Creation of blueprints would result in a blank red error dialog for the user and they would not be taken to the TaskPage

## Impact
All blueprints, except the tech-docs template, was impacted

## Detection
User-reported as an issue in the Chofer-Care channel

## Root Cause
There was a ':repo' query string parameter introduced for all templates, to support github actions creation for techdocs templates, however this caused all non-techdocs blueprints to be unable to navigate to the TaskPage since the 'repo' variable is undefined. 

Further, chofer/backstage does not gracefully handle routing failures, resulting in the 'blank' error message seen by the user.

## Resolution
Added conditional statements to a few key pages to ensure the repo variable is passed only when not undefined to the TaskPage navigation.

After initial fix, github actions was still not being kicked off for techdoc templates. React router doesn't support optional params, so two distinct routes were created, with and without the repo parameter, to redirect to the TaskPage. 

### Resolution Steps
Added a conditional if statement to blueprint routing to include the :repo query string in routing requests only when techdocs or infra-live templates are used. Conditional statements on the task page to attempt github action creation only if :repo is defined.

## Timeline
- 2/24 - 11AM - Issue identified
- 2/24 - 12:40PM - Identified as caused by recent release, impact is understood, root cause hypothesis formed
- 2/24 - 1PM - Plan communicated to Product Owner and key stakeholders, decision to attempt a fix in place
- 2/24 - 1:30PM - Investigation and working session spawned to test out RCA hypothesis and attempt a fix
- 2/24 - 3PM - Root cause confirmed as related to new :repo query string parameter, tests successful in non-prod (not all tests possible)
- 2/24 - 7PM - Fix pushed to Prod and testing showed blueprint errors were resolved but github actions were not automatically being created for techdoc templates
- 2/25 - 9:30AM - Investigation of github actions behavior, using breakpoints. Routing with repo passed in was failing, RCA identified
- 2/25 - 10:30AM - Fix deployed and tested in nonprod
- 2/25 - 11AM - Confirmed working and pushed to prod

## Action Items
|Item|Type|Owner|
|--|--|--|
|Gap in testing blueprints, need to flush out the synthetics test suite epic for portal|Prevent|Zareen+Fenil|
|Sonarqube test coverage for mono repo is missing|Reactive|Jayson|
|Ownership over team-specific features in portal|Prevent|Jayson|
|Identify Old/Legacy policies and infrastructure in use for portal and action items to follow|Reactive|Zareen|
|Import infrastructure management outside of IAC, and create IAC for missing components|Reactive|Zareen|
|Techdocs template missing in subprod, needs investigation|Reactive|Zareen|
|Build times for Portal are long, have a conversation with DevEx on Kaizen|Reactive|Fenil|
|Intermittent jenkins access issues, raise with DevEx|Reactive|Zareen|

## Lessons Learned

### What went well
- Quick reaction and identification of RCA
- Did not have to rollback, were able to fix in place
- Quick code fix
### What went wrong
- Build time is painful
- Not doing local testing for the 1st round of fix cost time

## Supporting Material

[Recreated error](../assets/24_02_23_blank-error.PNG)
[Error log in Datadog](../assets/24_02_23_datadog-error.PNG)