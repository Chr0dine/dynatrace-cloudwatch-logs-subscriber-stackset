# CloudWatch Logs → Dynatrace Subscriber (StackSet)

A CloudFormation template, deployed as a **StackSet**, that automatically subscribes CloudWatch log groups to the Dynatrace Firehose delivery stream when they are tagged for monitoring, and unsubscribes them when the tag is removed.

When someone applies the opt-in tag (for example `SendLogToDynatrace=true`) to a log group, CloudTrail records the call, an EventBridge rule picks it up, and a Lambda function subscribes **that log group**. Removing the tag (or changing its value) triggers the same function to remove the subscription again, so the tag stays the single source of truth. A daily full check also reviews every tagged log group, catching any that were skipped earlier. Log groups are only subscribed if they also carry the required business tags (`BU:ApplicationName`, `BU:GEARID`, `BU:SoftwareInstallationId`), matched regardless of capitalization.

You choose the organizational units (OUs) and regions when you deploy the StackSet. Each account and region gets one stack instance. When VPC networking is enabled, each instance finds its own subnets and security groups by tag, so one set of parameters works across accounts whose network IDs all differ.

## Files

| File | Purpose |
| --- | --- |
| `cloudwatch-logs-dynatrace-subscriber-stackset.yaml` | The CloudFormation template. Both Lambda functions' code is inline in this file, and this is the copy that gets deployed. |

## Deployment model: one stack instance per account and region

Each target account gets one Lambda per region, deployed only to the regions that have a Dynatrace Firehose stream. A single central Lambda per account was considered and rejected:

- **The trigger is regional.** CloudTrail events reach EventBridge only in the region where the tagging call was made. A central Lambda would still need a forwarding rule and an IAM role in every region, so it would swap the per-region resources rather than remove them.
- **The destination is regional.** A subscription filter must point at a Firehose stream in the same region as the log group.
- **Cross-region calls are fragile.** A Lambda in one region's VPC calling other regions' APIs needs NAT internet access; VPC interface endpoints only serve their own region.
- **Isolation.** A problem in one region (an outage, a broken subnet) doesn't stop log onboarding in the others.

The per-region copies cost essentially nothing: an idle Lambda, its EventBridge rules and its VPC network interfaces are free, and the number of runs is the same either way. The IAM roles are created per region too. IAM is global, so these are technically duplicates, but keeping them per stack instance means deleting one region's instance can never break another region.

Duplicates within an account are prevented by the fixed function name: a second stack instance in the same account and region (another StackSet, an overlapping OU, or a leftover standalone stack) fails on the name rather than creating a second subscriber.

## How it works

The function runs in one of two modes, chosen from how it was invoked.

**Targeted check:** triggered by tagging or untagging a log group:

```
Log group tagged / untagged
        │  (TagResource / TagLogGroup / CreateLogGroup,
        │   UntagResource / UntagLogGroup)
        ▼
CloudTrail ──► EventBridge rule ──► Subscriber Lambda
                                        │
                                        ├─ 1. Read the current tags of the log group named in the event
                                        ├─ 2. Not tagged TAG_KEY=TAG_VALUE any more?
                                        │       UNSUBSCRIBE_ON_TAG_REMOVAL=true  → delete the managed filter
                                        │       UNSUBSCRIBE_ON_TAG_REMOVAL=false → stop
                                        ├─ 3. Stop if a required BU: tag is missing or blank
                                        ├─ 4. Find the Dynatrace Firehose stream and IAM role (cached)
                                        └─ 5. Create or update that log group's subscription filter
```

The function always acts on the log group's **current** tags, not on the kind of event. If someone tags and then quickly untags a log group (or the reverse), both runs see the final state and agree.

**Full check:** triggered by the daily schedule, or by running the function manually with `{}`:

```
Daily schedule / manual run ──► Subscriber Lambda
                                    │
                                    ├─ 1. List every log group tagged TAG_KEY=TAG_VALUE
                                    ├─ 2. Skip any missing a required BU: tag
                                    ├─ 3. Find the Dynatrace Firehose stream and IAM role (always fresh)
                                    └─ 4. Create or update the subscription filter on the rest
```

Bulk tagging 200 log groups therefore produces 200 short targeted runs, each touching one log group, rather than 200 runs that each check everything. Both modes are idempotent: log groups that are already correctly subscribed are left alone.

### Network lookup (deploy time)

When **Run the function in a VPC** is `true`, a second, small Lambda runs once when each stack instance is created (and again when its VPC parameters change). It:

1. Finds every subnet tagged with the subnet tag key and value. All matches must be in one VPC, and there can be at most 16.
2. Finds the security groups **in that VPC** tagged with the security group tag key and value. There can be at most 5.
3. Hands the IDs to the subscriber function's VPC settings.

If nothing matches, or the matches break a rule above, the stack instance fails with a message saying which. The lookup runs outside any VPC, so it never depends on the network it is looking up, and it never runs per log event.

When the toggle is `false`, the lookup function and its role aren't created, and the subscriber runs outside any VPC using Lambda's own internet access.

### Matching rules

| What | How it's matched |
| --- | --- |
| Firehose stream | Name contains `FIREHOSE_NAME_CONTAINS` **and** the region (e.g. `us-east-1`), case-insensitive. Exactly one stream must match, and it must be `ACTIVE`. |
| CloudWatch Logs role | Name contains `ROLE_NAME_CONTAINS` **and** the region, case-insensitive. Its trust policy must allow `logs.amazonaws.com`. Exactly one role must match. |
| Opt-in tag | `TAG_KEY` and `TAG_VALUE` are matched exactly (case-sensitive) by the EventBridge rules and by the function. In a targeted check the function re-reads the log group's tags, so the latest change always wins. |
| Required tags | Each key in `REQUIRED_TAG_KEYS` must be present with a non-blank value. Key matching is case-insensitive, so `BU:GEARID`, `BU:gearid` and `bu:GearId` are all accepted. If a log group has several spellings of the same key, any one with a value satisfies it. |
| Subnets and security groups | Tag key matched exactly; tag value matched case-sensitively, with `*` and `?` as wildcards (EC2 filter rules). |
| Unsubscribe | Only the filter named `SUBSCRIPTION_FILTER_NAME` is removed. Any other subscription filters on the log group are left alone. |

### Results

Each run returns a summary with a `mode` field (`targeted` or `full`), counts per status, and a `details` entry per log group.

| Status | Meaning |
| --- | --- |
| `created` | A new subscription filter was added. |
| `updated` | The managed filter existed but pointed at the wrong stream, role or pattern, and was corrected. |
| `exists` | Already correct; nothing changed. |
| `unsubscribed` | Targeted check only: the log group is no longer opted in, and its managed filter was removed. |
| `missing_tags` | Skipped because one or more required tags are missing or blank. The log lists which ones. |
| `not_tagged` | Targeted check only: the log group isn't opted in and has no managed filter to remove (or no longer exists), or unsubscribing is turned off. Nothing to do. |
| `skipped` | Skipped because the log group already has two other subscription filters (the CloudWatch Logs limit). |
| `error` | An AWS error occurred for this log group. Other log groups are still processed. |

The function returns status `200` when there are no errors and `207` when some log groups errored.

## Resources created (per stack instance)

| Logical ID | Type | Purpose |
| --- | --- | --- |
| `SubscriberExecutionRole` | `AWS::IAM::Role` | Execution role for the subscriber (see [Permissions](#permissions)). The name is generated by CloudFormation, so instances in several regions don't clash. |
| `SubscriberFunction` | `AWS::Lambda::Function` | The subscriber function. Python 3.13, reserved concurrency of 1. Runs in the looked-up subnets when VPC networking is enabled. |
| `CloudTrailTagRule` | `AWS::Events::Rule` | Starts a targeted check when the opt-in tag is applied to a log group. |
| `CloudTrailTagRulePermission` | `AWS::Lambda::Permission` | Allows that rule to invoke the function. |
| `CloudTrailUntagRule` | `AWS::Events::Rule` | Starts a targeted check when the opt-in tag is removed or changed to another value. Only when `UNSUBSCRIBE_ON_TAG_REMOVAL` is `true`. |
| `CloudTrailUntagRulePermission` | `AWS::Lambda::Permission` | Allows that rule to invoke the function. Only when `UNSUBSCRIBE_ON_TAG_REMOVAL` is `true`. |
| `FullCheckRule` | `AWS::Events::Rule` | Starts the daily full check. Not created if the schedule parameter is empty. |
| `FullCheckRulePermission` | `AWS::Lambda::Permission` | Allows the schedule to invoke the function. Not created if the schedule parameter is empty. |
| `NetworkLookupRole` | `AWS::IAM::Role` | Execution role for the network lookup. Only when VPC networking is enabled. |
| `NetworkLookupFunction` | `AWS::Lambda::Function` | Deploy-time subnet and security group lookup (see [Network lookup](#network-lookup-deploy-time)). Only when VPC networking is enabled. |
| `NetworkLookup` | `Custom::NetworkLookup` | Runs the lookup and holds its results. Only when VPC networking is enabled. |

The template also has a **Rule** that runs when each stack instance is submitted, before anything is created: when VPC networking is enabled, all four tag parameters must be filled in.

**Outputs:** `FunctionName`, `FunctionArn`, `ExecutionRoleArn`, `CloudTrailTagRuleArn`, `CloudTrailUntagRuleArn` (when unsubscribing is on), `FullCheckRuleArn` (when the schedule is enabled), and `VpcId`, `SubnetIds`, `SecurityGroupIds` (when VPC networking is enabled). The VPC outputs show what the lookup found in that account and region.

## Parameters

All parameters are set once for the whole StackSet. They can be overridden per account or region with stack instance parameter overrides, but the tag-based network lookup means that usually isn't necessary.

### Function

| Parameter | Default | Notes |
| --- | --- | --- |
| Function name | `CloudWatch-Logs-Dynatrace-Subscriber` | Must be unique within each account and region. |
| Memory (MB) | `256` | 128 to 10240. |
| Timeout (seconds) | `300` | 3 to 900. Size this for the full check, which visits every tagged log group; targeted checks take seconds. |

### VPC

| Parameter | Default | Notes |
| --- | --- | --- |
| Run the function in a VPC | `true` | `false` runs the function outside any VPC and ignores the rest of this section. |
| Subnet tag key | none | Required when the toggle is `true`. Tag your **private** subnets with it, ideally two or more in different Availability Zones, in every account and region you deploy to. They need a route to a NAT gateway (or VPC endpoints). |
| Subnet tag value | none | Required when the toggle is `true`. Case-sensitive; `*` matches any value. |
| Security group tag key | none | Required when the toggle is `true`. The groups must allow outbound HTTPS (443). |
| Security group tag value | none | Required when the toggle is `true`. Case-sensitive; `*` matches any value. |
| Allow IPv6 traffic for dual-stack subnets | `false` | Set to `true` only if the subnets are dual-stack. |
| Network lookup version | `1` | Change to any new value to re-run the lookup after subnet or security group tags change (see [Re-running the network lookup](#re-running-the-network-lookup)). |

### Environment variables

| Parameter | Default | Notes |
| --- | --- | --- |
| `TAG_KEY` | none (required) | Opt-in tag key. Case-sensitive. |
| `TAG_VALUE` | `true` | Opt-in tag value. Case-sensitive. |
| `REQUIRED_TAG_KEYS` | `BU:ApplicationName,BU:GEARID,BU:SoftwareInstallationId` | Comma-separated. Case-insensitive keys, values must be non-blank. Leave empty to disable the check. |
| `FIREHOSE_NAME_CONTAINS` | none (required) | Text in the Dynatrace Firehose stream name. |
| `ROLE_NAME_CONTAINS` | none (required) | Text in the CloudWatch Logs → Firehose role name. Also scopes the function's `iam:PassRole` permission, which **is** case-sensitive, so enter it with the role name's exact capitalization. |
| `SUBSCRIPTION_FILTER_NAME` | `DynatraceManagedSubscription` | Name of the filter the function manages, and the only one it ever removes. |
| `FILTER_PATTERN` | empty | CloudWatch Logs filter pattern. Empty forwards all log events to Dynatrace. |
| `UNSUBSCRIBE_ON_TAG_REMOVAL` | `true` | `true` removes the managed filter when the opt-in tag is removed or changed to another value. `false` leaves existing subscriptions in place and doesn't create the removal rule. |
| `LOG_LEVEL` | `INFO` | Minimum level written to the function's own log (see [Logging](#logging)). Does not affect what is sent to Dynatrace. |

### Daily full check

| Parameter | Default | Notes |
| --- | --- | --- |
| Schedule (UTC) | `cron(0 6 * * ? *)` | EventBridge schedule expression; the default runs daily at 06:00 UTC. Any `cron(...)` or `rate(...)` expression works. Leave empty to disable the full check. |

## Permissions

### Subscriber

The execution role has the AWS managed policy `AWSLambdaExecute` plus one inline policy:

| Statement | Actions | Why |
| --- | --- | --- |
| `CloudWatchLogsDiscovery` | `logs:DescribeLogGroups`, `logs:DescribeSubscriptionFilters`, `logs:PutSubscriptionFilter`, `logs:DeleteSubscriptionFilter`, `logs:ListTagsForResource` | Read a log group's tags (targeted check) and manage its subscription filter. |
| `FirehoseDiscovery` | `firehose:ListDeliveryStreams`, `firehose:DescribeDeliveryStream` | Find the Dynatrace stream. |
| `RoleDiscovery` | `iam:ListRoles` | Find the CloudWatch Logs role. |
| `PassLogsRole` | `iam:PassRole` on `role/*<ROLE_NAME_CONTAINS>*` in this account | Hand that role to CloudWatch Logs when creating a subscription. |
| `ec2permissions` | `ec2:CreateNetworkInterface`, `ec2:DescribeNetworkInterfaces`, `ec2:DeleteNetworkInterface`, `ec2:DescribeSubnets`, `ec2:AssignPrivateIpAddresses`, `ec2:UnassignPrivateIpAddresses` | Required for a Lambda that runs in a VPC. Only included when VPC networking is enabled. |
| `FindResourcesByTag` | `tag:GetResources` | Find every tagged log group (full check). |

`AWSLambdaExecute` is broad: it allows all CloudWatch Logs actions and reading and writing objects in any S3 bucket. Expect a security review to ask about it.

### Network lookup

Only created when VPC networking is enabled. The role has `AWSLambdaBasicExecutionRole` (its own log group) plus `ec2:DescribeSubnets` and `ec2:DescribeSecurityGroups`. It can't change anything.

## Prerequisites

- **StackSets with service-managed permissions:** Trusted access for CloudFormation StackSets enabled in AWS Organizations, and the StackSet created from the management account or a registered delegated administrator. Service-managed StackSets **never deploy to the management account itself**; if it needs the subscriber, deploy the template there as a standalone stack.
- **CloudTrail:** A trail (an organization trail counts) recording write management events in every target region. Without it, tagging doesn't trigger the function; only the daily full check would subscribe log groups, and nothing would unsubscribe them.
- **Firehose and role:** Exactly one matching Firehose stream and exactly one matching CloudWatch Logs role in every target account and region (see [Matching rules](#matching-rules)). Only choose regions where these exist; anywhere else, every run fails.
- **Network (VPC networking enabled):** In every target account and region, private subnets in a single VPC with a NAT gateway route, and security groups allowing outbound 443, all tagged with the keys and values you enter. The function calls the CloudWatch Logs, Firehose, IAM and Resource Groups Tagging APIs.
- **No existing subscriber:** No Lambda with the same function name in any target account and region. Delete any standalone stacks of the earlier, non-StackSet version first, or the stack instance fails on the name. Remove any triggers attached to older copies so two subscribers don't run.
- **Concurrency:** Reserving 1 execution requires each target account to keep at least 100 unreserved in each region. The standard 1,000 limit is fine; new low-limit accounts are not.

## Deployment

The template is about 69 KB, over the 51,200-byte limit for passing a template inline, so the CLI must read it from S3. Console uploads handle this automatically.

### Console

1. In the management account (or delegated administrator), go to CloudFormation → **StackSets** → **Create StackSet**.
2. Choose **Service-managed permissions**, **Upload a template file**, and pick `cloudwatch-logs-dynatrace-subscriber-stackset.yaml`.
3. Enter a StackSet name and fill in the parameters.
4. Under **Deployment options**:
   - **Deployment targets:** choose **Deploy to organizational units** and enter the OU IDs.
   - **Automatic deployment:** enable it so accounts that later join those OUs get the subscriber automatically.
   - **Regions:** choose only the regions that have a Dynatrace Firehose stream.
5. Tick **I acknowledge that AWS CloudFormation might create IAM resources** and submit.

### CLI

```bash
# 1. Upload the template.
aws s3 cp cloudwatch-logs-dynatrace-subscriber-stackset.yaml \
  s3://<bucket>/cloudwatch-logs-dynatrace-subscriber-stackset.yaml

# 2. Create the StackSet (the template and parameters).
#    Add --call-as DELEGATED_ADMIN when running from a delegated administrator.
aws cloudformation create-stack-set \
  --stack-set-name cwlogs-dynatrace-subscriber \
  --template-url https://<bucket>.s3.<bucket-region>.amazonaws.com/cloudwatch-logs-dynatrace-subscriber-stackset.yaml \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false \
  --capabilities CAPABILITY_IAM \
  --parameters \
    ParameterKey=EnableVpc,ParameterValue=true \
    ParameterKey=SubnetTagKey,ParameterValue=<subnet-tag-key> \
    ParameterKey=SubnetTagValue,ParameterValue=<subnet-tag-value> \
    ParameterKey=SecurityGroupTagKey,ParameterValue=<sg-tag-key> \
    ParameterKey=SecurityGroupTagValue,ParameterValue=<sg-tag-value> \
    ParameterKey=TagKey,ParameterValue=SendLogToDynatrace \
    ParameterKey=FirehoseNameContains,ParameterValue=<text> \
    ParameterKey=RoleNameContains,ParameterValue=<ExactCaseText>

# 3. Deploy to the chosen OUs and regions.
aws cloudformation create-stack-instances \
  --stack-set-name cwlogs-dynatrace-subscriber \
  --deployment-targets OrganizationalUnitIds=ou-xxxx-aaaaaaaa,ou-xxxx-bbbbbbbb \
  --regions us-east-1 us-west-2 \
  --operation-preferences RegionConcurrencyType=PARALLEL,FailureToleranceCount=0,MaxConcurrentCount=5
```

To add more OUs or regions later, run `create-stack-instances` again with just the new targets. To remove them, use `delete-stack-instances` with `--no-retain-stacks`.

Start with a single test OU and region, confirm it works (see below), then widen the targets.

## After deploying

Do these in one target account and region first.

1. **Check the network lookup** (VPC networking enabled). The stack instance's outputs list the VPC, subnets and security groups found. Confirm they're the private ones you expected.
2. **Backfill.** Log groups that were tagged before the stack existed are picked up by the first daily full check. To do it immediately, run the function from the Lambda console's **Test** tab with `{}` as the event; the response should show `"mode": "full"` and the counts.
3. **Test the trigger.** On a test log group that has all three BU tags, apply the opt-in tag the way your teams normally do, for example:

   ```bash
   aws logs tag-resource \
     --resource-arn arn:aws:logs:REGION:ACCOUNT:log-group:NAME \
     --tags SendLogToDynatrace=true
   ```

   Within a few minutes, `/aws/lambda/<FunctionName>` should show `Targeted check: only processing ['NAME']` followed by `Created subscription on NAME`, and the log group's **Subscription filters** tab should list the filter.
4. **Test the removal.** Remove the tag again:

   ```bash
   aws logs untag-resource \
     --resource-arn arn:aws:logs:REGION:ACCOUNT:log-group:NAME \
     --tag-keys SendLogToDynatrace
   ```

   The function should log `Removed subscription from NAME`, and the filter should be gone.
5. **Test the tag check.** Repeat step 3 with a log group missing one BU tag. The function should log a warning naming the missing tag.

## Operations

### Logging

The subscriber logs to `/aws/lambda/<FunctionName>`. `LOG_LEVEL` is a minimum threshold:

| Setting | Writes |
| --- | --- |
| `DEBUG` | Everything, including request-level logging from the AWS SDK (very verbose). |
| `INFO` | Each log group processed, plus warnings and errors. |
| `WARNING` | Skipped log groups and errors. |
| `ERROR` | Errors only. |

The network lookup logs to its own `/aws/lambda/...NetworkLookupFunction...` log group, including a warning when all matching subnets are in one Availability Zone.

### Lookup caching

Finding the Firehose stream and IAM role means listing every delivery stream and every IAM role in the account, which is the slowest part of a run. Targeted checks cache both results for 15 minutes on the warm Lambda instance; with reserved concurrency of 1, queued runs from bulk tagging all share that instance. Full checks always look both up fresh and refresh the cache, and any run that ends with an error clears it. Unsubscribing doesn't need either lookup.

### Monitoring

If a run fails outright (for example, the Firehose stream can't be found), Lambda retries it twice and then drops the event. Per-log-group errors don't trigger a retry; they show up as `error` entries in that run's results. A CloudWatch alarm on the function's **Errors** metric is the simplest way to be told about outright failures. A log group whose targeted subscribe run was dropped is still picked up by the next daily full check; a dropped unsubscribe run is not (see [Limitations](#limitations)).

### Updating parameters

Update the StackSet with new parameter values; CloudFormation rolls the change out to every stack instance. Changing `TAG_KEY` or `TAG_VALUE` updates the function and all EventBridge rules.

### Re-running the network lookup

The lookup runs when a stack instance is created or when any VPC tag parameter changes. If you later retag subnets or security groups without changing the parameters, update the StackSet (or just the affected stack instances) with a new **Network lookup version**, for example `2`. Each instance then looks up its network again and moves the function if the results changed.

### Updating the code

The code that runs is the inline copy in the template's `SubscriberFunction` → `Code` → `ZipFile` block (and `NetworkLookupFunction` for the lookup). To change it, edit that block, then update the StackSet. Edits made directly in the Lambda console are overwritten on the next update.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Stack instance fails: `No subnet is tagged ...` | The subnet tag is missing in that account or region, or the key or value differs (both are case-sensitive) | Tag the subnets, then retry the stack instance. |
| Stack instance fails: `Subnets tagged ... are in more than one VPC` | The tag is on subnets in two VPCs | Narrow the tag to one VPC's subnets, or use a more specific value. |
| Stack instance fails: `No security group in vpc-... is tagged ...` | No tagged security group in the same VPC as the subnets | Tag a security group in that VPC. |
| Stack instance fails: `... Lambda accepts at most 16` / `at most 5` | Too many subnets or security groups match | Narrow the tags. |
| Stack instance fails on submit: `... tag key is required ...` | VPC networking is on but a tag parameter is blank | Fill in all four, or turn VPC networking off. |
| Stack instance fails: function already exists | A subscriber with the same name is already in that account and region | Delete the old stack or function, or use a different function name. |
| Tagging a log group doesn't invoke the function | No CloudTrail trail in the region; tag key or value capitalization doesn't match exactly; log group tagged through Tag Editor / Resource Groups Tagging API (recorded as a different CloudTrail event) | Confirm the trail; check the exact tag; check the rule's **Invocations** metric in EventBridge. The daily full check will still subscribe the log group. |
| Removing the tag doesn't unsubscribe | `UNSUBSCRIBE_ON_TAG_REMOVAL` is `false`; tag removed through Tag Editor / Resource Groups Tagging API; no CloudTrail trail | Check the parameter and the untag rule's **Invocations** metric. Remove the filter manually, or re-apply and remove the tag with `aws logs`. |
| Runs time out | A tagged subnet has no NAT route (for example, a public subnet), or the security group blocks outbound 443 | Fix the routes or security group, or retag and [re-run the network lookup](#re-running-the-network-lookup). |
| Some runs succeed and others time out | Mix of working and non-working subnets | Same as above; Lambda spreads runs across all subnets. |
| Full check times out | Too many tagged log groups for the timeout | Raise the Timeout parameter (up to 900 seconds). |
| `No Firehose delivery stream was found` / `Multiple Firehose delivery streams matched` | `FIREHOSE_NAME_CONTAINS` too narrow or too broad, the region isn't in the stream name, or the StackSet deployed to a region without a stream | Adjust the parameter, or remove that region's stack instances. |
| `No CloudWatch Logs subscription role was found` / `Multiple ... roles matched` | `ROLE_NAME_CONTAINS` too narrow or too broad, region missing from the name, or the role doesn't trust `logs.amazonaws.com` | Adjust the parameter or the role's trust policy. |
| `AccessDenied` on `iam:PassRole` | `ROLE_NAME_CONTAINS` capitalization differs from the actual role name | Re-enter it with the exact capitalization. |
| Log group reported as `missing_tags` | One or more BU tags missing or blank | Add the tags. The next daily full check subscribes it, or run the function manually with `{}`. |
| Log group reported as `not_tagged` | The opt-in tag was removed (or the log group deleted) and there was no managed filter to remove | Nothing to do, unless the tag removal was unintended. |
| Log group reported as `skipped` | It already has two other subscription filters | Remove one, or leave it out of Dynatrace. |
| `Could not find a log group in the CloudTrail event` warning | Unexpected event shape | The function falls back to a full check automatically. |
| Stack instance fails on `ReservedConcurrentExecutions` | Account concurrency limit too low in that region | Request a limit increase. |

## Limitations

- **Unsubscribing is event-driven only.** The daily full check adds missing subscriptions but doesn't look for subscriptions to remove: that would mean checking the filters on every log group in the account, every day. A removal missed by the untag rule (tag removed through Tag Editor or the Resource Groups Tagging API, CloudTrail gap, or a run dropped after retries) leaves the subscription in place. Re-apply and remove the tag with `aws logs` to retry it.
- **Removing a BU tag doesn't unsubscribe.** Only the opt-in tag controls removal. A subscribed log group that later loses a required BU tag stays subscribed.
- **Deleting a stack instance doesn't unsubscribe.** Existing subscription filters stay in place.
- **Late BU tags wait for the daily check.** Fixing a skipped log group's BU tags doesn't trigger a run; the next full check picks it up. Run the function with `{}` to do it sooner.
- **Tagging method matters for the instant triggers.** The rules match CloudWatch Logs' own tagging calls (`TagResource`, `TagLogGroup`, `CreateLogGroup` with tags, `UntagResource`, `UntagLogGroup`). Other tagging methods rely on the daily full check (and have no fallback for removal); test each method your teams use.
- **The network lookup doesn't follow tag changes on its own.** Retagging subnets or security groups takes effect after you [re-run the lookup](#re-running-the-network-lookup).
- **Public subnets aren't blocked.** Nothing stops a public subnet from carrying the subnet tag; runs there just time out.
- **Firehose or role changes can take up to 15 minutes** to reach targeted checks because of the lookup cache. The next full check, or any failed run, refreshes it immediately.
- **Runs are serialized.** Reserved concurrency of 1 means bulk tagging queues runs; each is short, so a few hundred clear in minutes.
- **Stack instance deletion can be slow** while Lambda releases its VPC network interfaces.
