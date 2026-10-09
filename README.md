# CloudWatch Logs → Dynatrace Subscriber (StackSet)

CloudFormation templates that automatically subscribes CloudWatch log groups to the Dynatrace Firehose delivery stream when they are tagged for monitoring, and unsubscribes them when the tag is removed.

When someone applies the opt-in tag (for example `SendLogToDynatrace=true`) to a log group, CloudTrail records the call, an EventBridge rule picks it up, and a Lambda function subscribes **that log group**. Removing the tag (or changing its value) triggers the same function to remove the subscription again, so the tag stays the single source of truth. A daily full check also reviews every tagged log group, catching any whose tagging event was missed.

There are two templates: a **StackSet** for rolling the subscriber out across many accounts, and a **standalone stack** for a single account, such as a central logging account. Most of this README describes the StackSet; the standalone stack is covered in [Standalone stack](#standalone-stack).

You choose the organizational units (OUs) and regions when you deploy the StackSet. Each account and region gets one stack instance. When VPC networking is enabled, each instance finds its own subnets and security groups by tag, so one set of parameters works across accounts whose network IDs all differ.

## Files

| File | Purpose |
| --- | --- |
| `cloudwatch-logs-dynatrace-subscriber-stackset.yaml` | The StackSet template. Both Lambda functions' code is inline in this file, and this is the copy that gets deployed. |
| `cloudwatch-logs-dynatrace-subscriber.yaml` | The standalone stack template, for one account at a time. The subscriber code is inline in this file. |
| `bu-tags/cloudwatch-logs-dynatrace-subscriber-bu-tags.yaml` | The standalone stack template with a required business-tag check: log groups are only subscribed if they also carry the `BU:` tags. See [Standalone stack with required BU tags](#standalone-stack-with-required-bu-tags). |

## Choosing a template

| Use | When |
| --- | --- |
| **StackSet** | Log groups live in many accounts across your AWS Organization, and each account (and region) has its own Dynatrace Firehose stream and CloudWatch Logs role. One deployment covers every account in the chosen OUs, including accounts that join them later. |
| **Standalone stack** | You already centralize CloudWatch logs in one logging account, so only that account needs the subscriber. Also use it for the Organizations management account (service-managed StackSets never deploy there), for an account outside AWS Organizations, or to try the subscriber out in one account before rolling out the StackSet. |

If log groups must also carry business tags before they're sent to Dynatrace, use the standalone stack in `bu-tags/` instead of the plain one.

Don't use both in the same account and region: the second deployment fails on the function name (see [Deployment model](#deployment-model-one-stack-instance-per-account-and-region)).

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
                                        ├─ 3. Find the Dynatrace Firehose stream and IAM role (cached)
                                        └─ 4. Create or update that log group's subscription filter
```

The function always acts on the log group's **current** tags, not on the kind of event. If someone tags and then quickly untags a log group (or the reverse), both runs see the final state and agree.

**Full check:** triggered by the daily schedule, or by running the function manually with `{}`:

```
Daily schedule / manual run ──► Subscriber Lambda
                                    │
                                    ├─ 1. List every log group tagged TAG_KEY=TAG_VALUE
                                    ├─ 2. Find the Dynatrace Firehose stream and IAM role (always fresh)
                                    └─ 3. Create or update the subscription filter on each one
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

The template is about 64 KB, over the 51,200-byte limit for passing a template inline, so the CLI must read it from S3. Console uploads handle this automatically.

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
3. **Test the trigger.** On a test log group, apply the opt-in tag the way your teams normally do, for example:

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
| Log group reported as `not_tagged` | The opt-in tag was removed (or the log group deleted) and there was no managed filter to remove | Nothing to do, unless the tag removal was unintended. |
| Log group reported as `skipped` | It already has two other subscription filters | Remove one, or leave it out of Dynatrace. |
| `Could not find a log group in the CloudTrail event` warning | Unexpected event shape | The function falls back to a full check automatically. |
| Stack instance fails on `ReservedConcurrentExecutions` | Account concurrency limit too low in that region | Request a limit increase. |

## Standalone stack

`cloudwatch-logs-dynatrace-subscriber.yaml` deploys the same subscriber as a normal CloudFormation stack in one account and region. Deploy one stack per region that has a Dynatrace Firehose stream. Everything in [How it works](#how-it-works), [Matching rules](#matching-rules), [Operations](#operations) and [Troubleshooting](#troubleshooting) applies, apart from the differences below.

### Differences from the StackSet

| | StackSet | Standalone stack |
| --- | --- | --- |
| Network | Subnets and security groups found by tag in each account, by a deploy-time lookup function | Subnet and security group IDs entered directly; no lookup function |
| No VPC | Set **Run the function in a VPC** to `false` | Leave **Subnets** and **Security groups** empty |
| Unsubscribe on tag removal | Yes (`UNSUBSCRIBE_ON_TAG_REMOVAL`) | No. Removing the opt-in tag leaves the subscription in place; delete the filter manually. There is no untag rule. |
| Results | Includes `unsubscribed` | No `unsubscribed` status; `not_tagged` means the log group is no longer opted in |
| Template size | About 64 KB (CLI must use S3) | About 47 KB (CLI can pass it inline) |

### Running with or without a VPC

- **In a VPC:** enter the private subnet IDs and the security group IDs, comma-separated. They must all be in one VPC; Lambda rejects a mix when the function is created. The same network requirements as the StackSet apply: a NAT gateway route (or VPC endpoints), and outbound HTTPS (443) allowed.
- **Outside any VPC:** leave both empty. The function uses Lambda's own internet access and its role gets no EC2 network-interface permissions. Use this in an account that has no VPC.

Fill in both or neither. Entering only one of them fails the stack on submit, before anything is created, with a message saying which is missing.

### Parameters

The **Function**, **Environment variables** and **Daily full check** parameters are the same as the StackSet's (see [Parameters](#parameters)), except that there is no `UNSUBSCRIBE_ON_TAG_REMOVAL`. The VPC section is:

| Parameter | Default | Notes |
| --- | --- | --- |
| Subnets (`SubnetIds`) | empty | Comma-separated private subnet IDs, ideally two or more in different Availability Zones. Empty runs the function outside any VPC. |
| Security groups (`SecurityGroupIds`) | empty | Comma-separated security group IDs in the subnets' VPC. Empty runs the function outside any VPC. |
| Allow IPv6 traffic for dual-stack subnets | `false` | Set to `true` only if the subnets are dual-stack. Ignored outside a VPC. |

### Prerequisites

The same as the StackSet's [Prerequisites](#prerequisites), for the one account and each region you deploy to, minus the StackSets and Organizations requirements: CloudTrail recording write management events, exactly one matching Firehose stream and CloudWatch Logs role, the network (if you use a VPC), no existing subscriber with the same function name, and at least 100 unreserved concurrent executions.

### Deploying

#### Console

1. In the target account and region, go to CloudFormation → **Stacks** → **Create stack** → **With new resources (standard)**.
2. Choose **Upload a template file** and pick `cloudwatch-logs-dynatrace-subscriber.yaml`.
3. Enter a stack name and fill in the parameters. Leave **Subnets** and **Security groups** empty to run outside a VPC.
4. Tick **I acknowledge that AWS CloudFormation might create IAM resources** and submit.
5. Repeat in each region that has a Dynatrace Firehose stream.

#### CLI

```bash
aws cloudformation deploy \
  --stack-name cwlogs-dynatrace-subscriber \
  --template-file cloudwatch-logs-dynatrace-subscriber.yaml \
  --region us-east-1 \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    TagKey=SendLogToDynatrace \
    FirehoseNameContains=<text> \
    RoleNameContains=<ExactCaseText> \
    SubnetIds=subnet-aaaa,subnet-bbbb \
    SecurityGroupIds=sg-cccc
```

Omit the `SubnetIds` and `SecurityGroupIds` lines to run outside a VPC. `aws cloudformation deploy` creates the stack the first time and updates it on later runs, keeping any parameters you don't pass. Run it once per region, changing `--region`.

### After deploying

Follow [After deploying](#after-deploying) steps 2 and 3 (backfill and test the trigger). Skip the network lookup check: the stack's VPC settings are exactly what you entered. Skip the removal test: the standalone stack doesn't unsubscribe.

### Updating

Update the stack with the edited template or new parameter values (`aws cloudformation deploy` again, or **Update** in the console). Switching between a VPC and no VPC is a normal stack update.

### Standalone stack with required BU tags

`bu-tags/cloudwatch-logs-dynatrace-subscriber-bu-tags.yaml` is the standalone stack with one addition: a log group is only subscribed if, as well as the opt-in tag, it carries every required business tag with a non-blank value. By default those are `BU:ApplicationName`, `BU:GEARID` and `BU:SoftwareInstallationId`. Everything else (VPC or no VPC, parameters, triggers, deployment) is the same as the plain standalone stack.

#### How the check works

After the function finds a log group tagged `TAG_KEY=TAG_VALUE` (in either a targeted or a full check), it compares the log group's tags with the `REQUIRED_TAG_KEYS` list before touching its subscription:

```
Tagged log group
    │
    ├─ 1. Every key in REQUIRED_TAG_KEYS present with a non-blank value?
    │       no  → skip it, report it as missing_tags, log which tags are missing
    │       yes ↓
    ├─ 2. Find the Dynatrace Firehose stream and IAM role
    └─ 3. Create or update the subscription filter
```

| Rule | Detail |
| --- | --- |
| Key matching | Case-insensitive, so `BU:GEARID`, `BU:gearid` and `bu:GearId` all satisfy `BU:GEARID`. If a log group has several spellings of the same key, any one with a value counts. |
| Values | Must be non-blank. A tag whose value is empty or only spaces counts as missing. Any non-blank value is accepted. |
| Which tags are read | In a targeted check, the log group's current tags, read fresh when the function runs. In a full check, the tags returned by the Resource Groups Tagging API search. |
| Disabling | Leave `REQUIRED_TAG_KEYS` empty to turn the check off, which makes the template behave like the plain standalone stack. |

#### Extra parameter

| Parameter | Default | Notes |
| --- | --- | --- |
| `REQUIRED_TAG_KEYS` | `BU:ApplicationName,BU:GEARID,BU:SoftwareInstallationId` | Comma-separated tag keys. Case-insensitive keys, values must be non-blank. Blank entries and duplicates are ignored. Leave empty to disable the check. |

#### Extra result status

| Status | Meaning |
| --- | --- |
| `missing_tags` | Skipped because one or more required tags are missing or blank. The run's `details` entry lists them under `missingTags`, and the function logs a warning naming them. |

#### Deploying

Follow [Deploying](#deploying) for the standalone stack, picking `bu-tags/cloudwatch-logs-dynatrace-subscriber-bu-tags.yaml`. This template is about 51.6 KB, just over the 51,200-byte limit for passing a template inline, so the CLI needs an S3 bucket to stage it (the console handles this automatically):

```bash
aws cloudformation deploy \
  --stack-name cwlogs-dynatrace-subscriber \
  --template-file bu-tags/cloudwatch-logs-dynatrace-subscriber-bu-tags.yaml \
  --s3-bucket <bucket-in-the-same-region> \
  --region us-east-1 \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    TagKey=SendLogToDynatrace \
    FirehoseNameContains=<text> \
    RoleNameContains=<ExactCaseText> \
    SubnetIds=subnet-aaaa,subnet-bbbb \
    SecurityGroupIds=sg-cccc
```

`RequiredTagKeys` keeps its default unless you add `RequiredTagKeys=<key1>,<key2>` to the overrides.

#### Testing the check

After the [standard checks](#after-deploying-1), tag a test log group that has all the required tags; it should be subscribed. Then tag one that's missing a `BU:` tag; the function should log `Skipping NAME because these required tags are missing or empty: [...]` and report it as `missing_tags`.

#### Behaviour to know about

- **Apply the BU tags before (or with) the opt-in tag.** If the opt-in tag goes on first, that run skips the log group, and adding the BU tags afterwards doesn't trigger another run. The next daily full check subscribes it; run the function with `{}` to do it sooner.
- **Removing a BU tag doesn't unsubscribe.** A subscribed log group that later loses a required tag stays subscribed.

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Log group reported as `missing_tags` | One or more required tags missing or blank | Add the tags. The next daily full check subscribes it, or run the function manually with `{}`. |

## Limitations

- **Unsubscribing is event-driven only.** The daily full check adds missing subscriptions but doesn't look for subscriptions to remove: that would mean checking the filters on every log group in the account, every day. A removal missed by the untag rule (tag removed through Tag Editor or the Resource Groups Tagging API, CloudTrail gap, or a run dropped after retries) leaves the subscription in place. Re-apply and remove the tag with `aws logs` to retry it.
- **Deleting a stack instance doesn't unsubscribe.** Existing subscription filters stay in place.
- **Tagging method matters for the instant triggers.** The rules match CloudWatch Logs' own tagging calls (`TagResource`, `TagLogGroup`, `CreateLogGroup` with tags, `UntagResource`, `UntagLogGroup`). Other tagging methods rely on the daily full check (and have no fallback for removal); test each method your teams use.
- **The network lookup doesn't follow tag changes on its own.** Retagging subnets or security groups takes effect after you [re-run the lookup](#re-running-the-network-lookup).
- **Public subnets aren't blocked.** Nothing stops a public subnet from carrying the subnet tag; runs there just time out.
- **Firehose or role changes can take up to 15 minutes** to reach targeted checks because of the lookup cache. The next full check, or any failed run, refreshes it immediately.
- **Runs are serialized.** Reserved concurrency of 1 means bulk tagging queues runs; each is short, so a few hundred clear in minutes.
- **Stack instance deletion can be slow** while Lambda releases its VPC network interfaces.
- **The standalone stack doesn't unsubscribe.** Removing the opt-in tag leaves the subscription filter in place. Use the StackSet template if you need removal.
