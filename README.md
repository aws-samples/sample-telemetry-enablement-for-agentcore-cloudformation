# bedrock-agentcore-telemetry-enablement

This CloudFormation template turns on CloudWatch Observability telemetry
(logs and traces) for Amazon Bedrock AgentCore resources. 

## Prerequisites

- An AWS account with Bedrock AgentCore available in your target region.
- AWS CLI v2 configured with credentials for that account and region.
- Permissions to deploy the stack, including creating an IAM role
  (`CAPABILITY_IAM`), a Lambda function, ObservabilityAdmin telemetry rules, and
  an X-Ray resource policy (`xray:PutResourcePolicy`).

### One-time observability setup (per account and region)

Enable the following in each region where you deploy this stack. These are
account and region level settings and are not created by this template.

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
| CodeInterpreter | `AWS::ObservabilityAdmin::TelemetryRule` | Usage logs, traces |
| Gateway | `Custom::TelemetryRule` | Application logs, traces |
| Memory | `Custom::TelemetryRule` | Application logs, traces |
| WorkloadIdentity | `Custom::TelemetryRule` | Application logs, traces |

Gateway, Memory, and WorkloadIdentity are not yet in the CloudFormation schema,
so they are created through an inline Python Lambda that calls the
`observabilityadmin` API and signals CloudFormation using the official
`cfnresponse` module. Create must succeed for every enabled rule; any error
(including a conflict with an existing rule) is reported as a failure so the
stack surfaces it. On delete, a rule that is already gone is treated as success.

### Resources created

- Native and custom telemetry rule resources for each enabled resource type
  (scoped to the stack region).
- `AgentCoreXRayResourcePolicy` - an `AWS::XRay::ResourcePolicy` that authorizes
  the log delivery service to write AgentCore traces to X-Ray in this region.
  Created only when at least one resource type is enabled.
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
