# Enable Amazon Bedrock AgentCore Observability with CloudFormation

Deploy a single CloudFormation stack to turn on CloudWatch logs and X-Ray traces
for every Amazon Bedrock AgentCore resource type: Runtime, Gateway, Memory,
Browser, CodeInterpreter, and WorkloadIdentity.

Uses `AWS::ObservabilityAdmin::TelemetryRule` where the CloudFormation schema
supports it, and the `observabilityadmin` API through a Lambda custom resource
where it does not.


## Prerequisites

- An AWS account with Bedrock AgentCore available in your target region.
- AWS CLI v2 (2.36 or newer) configured with credentials for that account and region. Version 2.36+ includes the `observabilityadmin` telemetry rule operations this stack relies on. On earlier versions, those commands return an "invalid choice" error.
- Permissions to deploy the stack, including creating an IAM role
  (`CAPABILITY_IAM`), a Lambda function, ObservabilityAdmin telemetry rules, and
  an X-Ray resource policy (`xray:PutResourcePolicy`).

### One-time observability setup (per account and region)

Enable the following in each region where you deploy this stack. These are
account and region level settings and are not created by this template.

- **Telemetry Evaluation** (required for the telemetry rule APIs). Enable once
  per account: `aws observabilityadmin start-telemetry-evaluation --region <your-region>`.
  Without it, rule operations fail with a `ValidationException`.

- **CloudWatch Transaction Search** (required for traces). AgentCore trace
  delivery to X-Ray depends on Transaction Search being enabled; without it,
  trace deliveries fail. This is a one-time setup per account and region, via the
  CloudWatch console or the API.
  - [Enable Transaction Search](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Enable-TransactionSearch.html)

- **CloudWatch telemetry resource discovery** Enabling telemetry
  configuration lets CloudWatch discover your AWS resources and their telemetry
  configuration metadata (it creates AWS Config service-linked configuration
  recorders). This powers the CloudWatch telemetry overview and helps you see
  which resources have telemetry enabled.
  - [What is telemetry discovery and enablement?](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/telemetry-config-what-is.html)
  - [Setting up telemetry configuration (Enable Resource Discovery)](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/telemetry-config-turn-on.html)

## What it deploys

The template creates telemetry enablement rules that apply to all current and
future AgentCore resources of each enabled type in the region. Logs are delivered
to CloudWatch Logs under the pattern
`/aws/bedrock-agentcore/<resourceId>/<logType>`. Traces are delivered to X-Ray
and ingested into the CloudWatch Logs group `aws/spans`.

Two delivery paths are used, depending on CloudFormation schema support:

| Resource type | Path | Telemetry |
|---|---|---|
| Runtime | `AWS::ObservabilityAdmin::TelemetryRule` | Application logs, usage logs, traces |
| Browser | `AWS::ObservabilityAdmin::TelemetryRule` | Usage logs, traces |
| CodeInterpreter | `AWS::ObservabilityAdmin::TelemetryRule` | Application logs, usage logs, traces |
| Gateway | `Custom::TelemetryRule` | Application logs, traces |
| Memory | `Custom::TelemetryRule` | Application logs, traces |
| WorkloadIdentity | `Custom::TelemetryRule` | Application logs, traces |

Gateway, Memory, and WorkloadIdentity are not yet in the CloudFormation schema,
so they are created through an inline Python Lambda that calls the
`observabilityadmin` API and signals CloudFormation using the official
`cfnresponse` module. Create must succeed for every enabled rule; any error
(including a conflict with an existing rule) is reported as a failure so the
stack surfaces it. On delete, a rule that is already gone is treated as success.

### Payload logging and sensitive content

Read this before deploying. Application logs for several AgentCore resource
types record request and response content so data that passes through your
agents is written to your log destination. Treat these logs as sensitive and
do not put credentials, secrets, tokens, or keys in prompts, tool arguments, or
code that CodeInterpreter runs.

| Resource type | What the application logs record |
|---|---|
| Runtime | The request payload of each agent invocation, such as user prompts. The agent response is not recorded. |
| CodeInterpreter | The full request and response payload of each `InvokeCodeInterpreter` call, including the code and its output. |
| Gateway | The request and response bodies of MCP calls, including tool names, tool arguments, and tool results. |
| WorkloadIdentity | The request and response payloads of Identity operations such as `GetWorkloadAccessToken`. Responses from these operations return access tokens. Treat these logs as credential-bearing unless you have confirmed the tokens are not recorded. |
| Memory | Extraction and consolidation processing events with actor, session, and namespace identifiers. AWS documentation does not list payload fields. |
| Browser | Not applicable. This template enables only usage logs and traces for Browser, which do not contain request content. |

The rules apply to all current and future resources of each enabled type in
the account and region. This template only turns telemetry on. It does not
redact payloads or secure the logs. You are responsible for the following on
each destination log group.

- Read access, including subscription filters and exports to Amazon S3 or
  Firehose.
- Retention.
- Encryption at rest, for example a customer-managed KMS key.

If payload capture is not acceptable for a resource type, set its
`Enable*Telemetry` parameter to `false` before deploying.

### Resources created

- Native and custom telemetry rule resources for each enabled resource type
  (scoped to the stack region).
- `AgentCoreXRayResourcePolicy` - an `AWS::XRay::ResourcePolicy` that authorizes
  the log delivery service to write AgentCore traces to X-Ray in this region.
  Created only when at least one resource type is enabled. Required for Browser and CodeInterpreter traces since their ARNs are AWS-owned and the service does not auto-create a policy for them. Do not remove it.
- `TelemetryRuleFunction` - inline Python 3.13 Lambda (custom resource handler),
- `TelemetryRuleFunctionRole` - IAM execution role for the Lambda.

## Architecture

![Architecture diagram](architecture.png)

Runtime, Browser, and CodeInterpreter are created directly by CloudFormation as native
`AWS::ObservabilityAdmin::TelemetryRule` resources. Gateway, Memory, and
WorkloadIdentity are created through the inline Lambda custom resource. The
stack also creates the X-Ray resource policy that authorizes trace delivery in
the region.

## Parameters

Each `Enable*Telemetry` parameter controls whether the logging and tracing
telemetry rules for one resource type are created. All default to `"true"`, so a
plain deploy enables telemetry for every resource type in the stack's region.

| Parameter | Default | Allowed values | Effect |
|---|---|---|---|
| `EnableRuntimeTelemetry` | `true` | `true`, `false` | Create logging and tracing rules for Runtime resources |
| `EnableBrowserTelemetry` | `true` | `true`, `false` | Create logging and tracing rules for Browser resources |
| `EnableCodeInterpreterTelemetry` | `true` | `true`, `false` | Create logging and tracing rules for CodeInterpreter resources |
| `EnableGatewayTelemetry` | `true` | `true`, `false` | Create logging and tracing rules for Gateway resources |
| `EnableMemoryTelemetry` | `true` | `true`, `false` | Create logging and tracing rules for Memory resources |
| `EnableWorkloadIdentityTelemetry` | `true` | `true`, `false` | Create logging and tracing rules for WorkloadIdentity resources |

There is no region parameter. The stack acts on the region it is deployed to
(`AWS::Region`).

## Deploy

```bash
aws cloudformation deploy \
  --template-file bedrock-agentcore-telemetry-template.yaml \
  --stack-name bedrock-agentcore-telemetry-enablement \
  --capabilities CAPABILITY_IAM \
  --region <your-region>
```

To disable a resource type, pass its parameter as `false`:

```bash
aws cloudformation deploy \
  --template-file bedrock-agentcore-telemetry-template.yaml \
  --stack-name bedrock-agentcore-telemetry-enablement \
  --capabilities CAPABILITY_IAM \
  --region <your-region> \
  --parameter-overrides EnableBrowserTelemetry=false EnableCodeInterpreterTelemetry=false
```

`CAPABILITY_IAM` is required because the template creates an IAM role for the
Lambda.

### Validate before deploying

```bash
cfn-lint bedrock-agentcore-telemetry-template.yaml
aws cloudformation validate-template --template-body file://bedrock-agentcore-telemetry-template.yaml
```

### Confirm the rules were created

```bash
aws observabilityadmin list-telemetry-rules --region <your-region>
```

The rules create deliveries for matching resources automatically. Log
deliveries usually appear within a few minutes. Trace deliveries for the
AWS-provided Browser and CodeInterpreter can take longer, so wait several
minutes before you conclude that traces are not being delivered.

### Upgrading a stack deployed from an earlier version

Earlier versions of this template created separate application-log and
usage-log rules for Runtime. CloudWatch allows only one Logs rule for each
resource type and destination, so this version uses a single rule that
requests both log types. An in-place stack update from an earlier version
fails, because the new combined rule conflicts with the old rule that is still
present during the update.

To upgrade, delete the existing stack and then deploy this version. Deleting
the stack does not remove existing deliveries, so telemetry that is already
flowing keeps flowing while you redeploy.

```bash
aws cloudformation delete-stack --stack-name bedrock-agentcore-telemetry-enablement --region <your-region>
aws cloudformation wait stack-delete-complete --stack-name bedrock-agentcore-telemetry-enablement --region <your-region>
```

Then run the deploy command above.

## Teardown and cleanup

Turning telemetry off is not the reverse of turning it on. Deleting a rule,
setting an `Enable*Telemetry` parameter to `false`, or deleting the stack has
the following effects. See the
[DeleteTelemetryRule API reference](https://docs.aws.amazon.com/cloudwatch/latest/observabilityadmin/API_DeleteTelemetryRule.html)
for the service behavior.

- Telemetry is no longer enabled on newly discovered resources.
- Existing deliveries, delivery sources, and delivery destinations remain so
  data that is already flowing keeps flowing.
- Log groups and stored log events and traces remain.
- The `AWSLogDeliveryAgentCoreXRay` resource policy remains until you delete
  the whole stack because it is a stack resource.

### Targeted cleanup

Remove only what this stack's rules created in this account and do not delete
shared destinations. In particular, other AgentCore resources and stacks may
write traces to the `aws/spans` log group. Leave it in place unless you have
confirmed nothing else uses it.

1. List deliveries, delivery sources, and delivery destinations. Identify
   the ones created for the AgentCore resource types you enabled, for example
   by names that match those resource types.

   ```bash
   aws logs describe-deliveries --region <your-region>
   aws logs describe-delivery-sources --region <your-region>
   aws logs describe-delivery-destinations --region <your-region>
   ```

2. Delete those, after confirming they are not shared.

   ```bash
   aws logs delete-delivery --id <delivery-id> --region <your-region>
   aws logs delete-delivery-source --name <source-name> --region <your-region>
   aws logs delete-delivery-destination --name <destination-name> --region <your-region>
   ```

3. If you no longer need the stored telemetry, delete the per-resource log
   groups under `/aws/bedrock-agentcore/<resourceId>/<logType>`. Deletion is
   permanent so review the contents first.

   ```bash
   aws logs describe-log-groups --log-group-name-prefix /aws/bedrock-agentcore/ --region <your-region>
   ```

### Verify the cleanup

Confirm that no rules or unexpected delivery configuration remain.

```bash
aws observabilityadmin list-telemetry-rules --region <your-region>
aws logs describe-delivery-sources --region <your-region>
aws logs describe-delivery-destinations --region <your-region>
```

If a rule delete fails, the inline custom resource reports the failure and the
stack can end up in `DELETE_FAILED`. Fix the underlying error or remove the
rule with `aws observabilityadmin delete-telemetry-rule` then retry the stack
deletion.

## Security

This solution is provided as a proof-of-value and is not intended as a
production-ready implementation. You are responsible for evaluating how the AWS
Shared Responsibility Model applies to your specific use case and for
implementing the necessary security controls to meet your organization's
requirements. AWS provides a broad set of security tools and configurations to
support your security objectives, and it is your responsibility as the developer
to ensure all aspects of your application are appropriately secured for
production use.

## License

See the LICENSE file for licensing information.
