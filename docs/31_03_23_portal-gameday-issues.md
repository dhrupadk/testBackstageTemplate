# Postmortem - Portal Nonprod Destroyed and unable to recreate

## Incident Date
03/29/23

## Authors
Fenil, Mitch

## Status
Complete, Action Items pending

## Summary
Team ran a gameday involving destruction of chofer nonprod, post-gameday was unable to bring the environment back up

## Impact
No real impact to end-users, dev team impact and unable to push code without a nonprod environment, sprint points were halted

## Detection
Found by team at the end of the gameday

## Root Cause
- Resources created & managed by chofer-dev terraform had been manually leveraged by other services in the same account
- Lack of knowledge in execution and state of the backing terraform for chofer-dev
- Multiple tech debt items impacting resolution, such as:
  - Legacy terraform module use
  - Missing IaC for features such as OpenSearch and Techdocs S3 bucket
  - Lack of infra-live structure
  - Portal pipelines are slow to execute
  - Intervowen portal pipeline execution
  - Lack of seperation between baseline platform insfrastructure and app infrastructure

## Resolution
- Use infra-live repo to shorten cycle time
- Creating a fresh environment from a blank slate by updating service naming 
- Add manual missing permissions

### Resolution Steps
- Attempts to fix in place
- Created up an alternate pipeline & infra-live repo setup to be able to iterate significantly faster than the normal pipeline
- Deleted key conflicting resources by hand caused by 'fix in place' attempts earlier
- Spun up an environment using existing TF, with updated name_prefix, hardcoded artifact version, and a fresh state bucket
- Reconciliation of new repo with existing TF to ensure developer's ability to use old pipelines for further work

## Timeline
- 3/29 Morning - Applied empty TF module to undo all as a part of gameday (feature/game-day-destroy)
- 3/29 Evening - Concluded gameday, attempted to restore infrastructure to old state based on IaC, didn't work. Houston, we have a problem.
- 3/30 Morning/Afternoon - War room call scheduled to attempt to rectify. Team attempted to resolve errors in the Terraform Apply one at a time, such as resources that could not be deleted due to being shared, etc. Identified use of 'old' mgmt vpc vs regular standard vpc and other debt items impacting the situation
- 3/30 Evening - Continued attempted to clean up state and delete specific resources. Started to rename resources to try to work around deletions & state issues. Problem resources including WAF, SGs, ACM cert (shared with chofer-uat), VPC Endpoints (sharing rds SGs), and more were shared by other products or environments, impacting ability to destroy or clean up these resources. E.g. Had 3 databases to clean up manually and ECS clusters for chofer-uat that were not in use and living in a half-built state.
- 3/30 Late Evening - Team encountered continued DevEx issues (4+ issues observed over different attempts), pipeline failures on execution unrelated to portal code or TF. Errors throw by runners related to jenkins issues. 
- 3/31 Morning - Attempted to execute state with name_prefix changes across everything (e.g. DNS), still not working 
- 3/31 Late Morning - Team pivoted to creating a infra-live-_esque_ version of the chofer-dev terraform deployment (using the existing main.tf) to avoid the flakyness and long-cycle times of the normal self-service-mono pipeline. Testing any changes to the portal took 60m and since the team had to wait for the full build before TF apply/plan would occur. Further, alongside the new repo, the team ensured use of a new state bucket with no state impacts from all previous resolutions attempts
- 3/31 Afternoon - Continued iteration on infra-live / main.tf hybrid approach. Found out while starting new from fresh state, previous name_prefix changes resulted in conflicting resources that had been partially deployed to AWS, had to delete manually. (2 ECR repos and more). Fixing a bunch of TF issues in the new hybrid for a few iterations. 
- 3/31 Afternoon - Artifactory connection issues once the initial conflicts were resolved and infra-live setup was working. Brought in devex, learned we could use non-deployer terraformStandalone which could connect to artifactory when deployer-pattern setup could not (which was blocking this approach from fixing the issue). Added params for hardcoded artifact_name, artifact_path, image_tag to get this working with this non-deployer terraformStandalone pipeline.
- 3/31 Afternoon - Added ace- prefix to name_prefix everywhere due to more state issues, WAF 'rule does not exist' something (more issues from previous fix in place attempts)
- 3/31 Afternoon - Ran and applied successfully. 
- 3/31 Evening - Chofer infra is live and page loads, cannot login. Team updated the redirect URI in AzureAD to reflect new DNS, login worked. 
- 3/31 Evening - Few features broken in the portal, due to permissions. Manually added of policies for s3 / opensearch as these were not present in the existing main.tf. Opensearch still failing, deemed not req'd for now, to be fixed later. Insights/FinOps apis still broken, to be fixed later. 
- 3/31 Evening - Attempted to trigger infra-live type build from the self-service-mono repo now (deployTfArtifact), with now an updated tf state bucket to ensure use of the now-valid TF state with the unchanged main.tf code. Added 'waitforTFapproval=true' to test an actual plan on old repo, this worked, confirming that the team could use the old pipeline with the new state and name_prefix setup we have used to create the new environment.

## Action Items

|Item|Type|Owner|Priority|
|--|--|--|--|
|Insights API separation (public/private) needs investigation and addressing from an architecture POV| Remediation | Jayson | Critical |
|FinOps integration is broken, chofer-dev cannot connect to the existing APIs that are in the mgmt vpc|Remediation| Zareen | Critical |
|Search is broken (doesn't exist) in chofer-dev, needs IaC and deployment | Remediation | Zareen | Critical |
|Do not destroy before creating during game-days | Preventative | Edson | Critical |
|Techdocs s3 bucket needs backing IaC| Remediation | Zareen | High |
|Manual IAM additions need backing IaC and to be deployed properly | Remediation | Zareen | High |
|Portal IaC (main.tf and more) needs upgrades to latest module versions| Remediation | Zareen | High |
|Needs rollout to production, same conflicts and shared resources exist there| Remediation | Zareen | High|
|Portal pipelines need optimization (60m way too long)| Preventative | Zareen (+devex) | High |
|Portal needs a permanent infra-live setup (app artifacts separate from baseline infra)| Preventative | Zareen | Medium |
|Chofer portal needs a 'seeding' script to spawn the initial dataset| Preventative | Zareen | Medium |
|Need to migrate everything (all products including portal) to standard vpc / new account| Preventative | Edson | Medium |
|Jenkins unavailability investigation and followup (or publishing of maintenance window) | Preventative | Mitchell | Medium |
|No Datadog hookup for chofer non-prod | Remediation | Fenil | Medium |
|Nonprod needs a housecleaning of any lagging/extraneous resources that were left over | Remediation | Edson | Low |

## Lessons Learned

### What went well
- Did a gameday
- Recovered, and made it a priority
- Reducing cycle time helped a lot
- Good lessons learned and actions (for the next gameday as well as the portal itself)
- Greater knowledge of infrastructure of portal

### What went wrong
- the gameday (lack of backup plan)
- impact to team of a last minute critical issue
- pipelines
- don't destroy before create, unplanned chaos test
- lack of understanding of existing infrastructure setup
- no DR plan
