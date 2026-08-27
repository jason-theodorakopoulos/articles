# Microsoft Agent 365 Custom Agent Registration and Governance Guidance

Last validated: 2026-08-27

This document gives general technical guidance for registering, governing, and operating custom-built agents with Microsoft Agent 365. It applies to agents built with frameworks and platforms such as LangGraph, LangChain, Microsoft Agent Framework, Microsoft 365 Agents SDK, OpenAI Agents SDK, Microsoft Foundry, and custom in-house runtimes.

Microsoft Agent 365 is the enterprise control plane for agents. It does not replace an agent runtime or orchestration framework. A custom agent can keep its own runtime, model provider, tool orchestration, and hosting model, while Agent 365 adds registry visibility, identity, policy, observability, tool governance, and security/compliance integration.

## Core Concepts

Agent 365 integration is best understood as a set of incremental capabilities:

- Register: make the agent visible and manageable in Microsoft 365 admin center.
- Identity: give the agent a governed Microsoft Entra identity model.
- Observability: emit auditable traces for invocations, LLM calls, tool calls, errors, and outputs.
- Work IQ and MCP tooling: route Microsoft 365 and business tool access through governed MCP servers where appropriate.
- Governance: apply admin controls for availability, deployment, permissions, policies, risk review, lifecycle, and ownership.
- AI teammate, where applicable: create an agent that operates with its own Microsoft 365 user-like identity, mailbox, Teams presence, and directory entry. This is currently tied to the Frontier preview program and has additional constraints.

## Registration Paths

Use the path that matches how the agent is built and hosted.

### Microsoft-native agents

Agents built with Microsoft Copilot Studio, Microsoft 365 Copilot Agent Builder, SharePoint, Microsoft 365 Agents Toolkit, and Microsoft Foundry have more built-in integration with Agent 365.

For example, Copilot Studio agents automatically appear in the Agent 365 registry and emit telemetry to the Agent 365 observability backend. Pro-code Microsoft 365 custom engine agents are discoverable through their existing Microsoft Entra app registration, and can add Agent 365 observability or other capabilities as needed.

### External platform registry sync

Registry sync can synchronize supported external platform agents into the Microsoft 365 admin center agent registry for centralized visibility and governance.

Currently documented supported platforms include:

- Amazon Bedrock
- Google Vertex AI
- Salesforce Agentforce
- Databricks Genie

Registry sync is a preview feature. Treat it primarily as inventory and governance integration. Deeper behavior visibility, Work IQ tool access, and end-to-end traceability may still require SDK, OpenTelemetry, or platform-specific integration.

### Custom pro-code agents

For self-hosted or externally hosted custom agents, use the Agent 365 SDK and CLI. This includes agents built with LangGraph, LangChain, Microsoft Agent Framework, OpenAI Agents SDK, Semantic Kernel, or custom Python/Node/.NET services.

The Agent 365 SDK does not create or host the agent. It layers enterprise capabilities on top of the existing agent implementation.

### What SDK integration means

For a custom agent, the implementation remains a three-layer stack:

1. The chosen LLM orchestrator and runtime, such as LangGraph or Microsoft Agent Framework.
2. The agent's prompts, workflows, tools, and application code.
3. Agent 365 enterprise capabilities, such as identity, registration, observability, notifications, and governed tooling.

Adding Agent 365 is therefore an augmentation rather than a runtime rewrite. Each enterprise capability is still a deliberate integration step, however, and most steps must be completed and validated for each agent.

Identity and registration are separate from observability. Registration creates an inventory record and governed identity, but it does not emit invocation, model, tool, error, or output activity. As a result, registration alone does not populate behavioral monitoring or reporting. Defender and Purview features that depend on runtime activity also require appropriately instrumented telemetry. Identity-, inventory-, and policy-based controls can apply before observability is added, but they do not replace it.

## Developer Effort and Sizing Model

Estimate custom-agent onboarding as a set of capability tiers rather than a single registration task.

| Tier | Work package | Primary owner | What it unlocks | Main implementation work |
| --- | --- | --- | --- | --- |
| 0 | Tenant enablement and CLI bootstrap | Global Administrator and platform team | Access to downstream Agent 365 setup | Confirm licensing and tenant enablement, install the CLI prerequisites, create the Agent 365 CLI enterprise application, and grant required admin consent. This is generally a one-time tenant activity. |
| 1 | Identity and registration blueprint | Agent developer with identity/admin support | Registry presence, Microsoft Entra Agent ID, and a target for governance and Conditional Access | Create the blueprint, select OBO/S2S identity modes, request least-privilege permissions and consent, then package and publish the agent. This is per agent. |
| 2 | Observability | Agent developer with operations/security support | Activity data for monitoring, reporting, and activity-dependent security/compliance investigation | Add the SDK or OpenTelemetry integration, acquire scoped tokens, instrument agent/model/tool/output/error paths, propagate trace context, control sensitive data, export traces, and validate ingestion. This is per agent and runtime. |
| 3 | Governed tools and Work IQ, when required | Agent developer and administrator | Governed Microsoft 365 or business-tool access through MCP | Configure tool manifests, authentication, permissions, consent, SDK adapters, and tool-call validation. This is capability-dependent. |
| 4 | Operational governance | Agent owner, platform, security, and compliance teams | Production policy, risk management, and lifecycle controls | Pilot the agent, apply access and security policies, validate Defender/Purview behavior, define support and retention, and test blocking and retirement. |

The practical sizing boundary for "integrate the SDK and add observability" is normally tiers 1 and 2, with tier 0 as a prerequisite and tiers 3 and 4 added according to scope. A registration-only estimate covers inventory and identity, not a monitored production integration.

Separate the estimate into:

- One-time per tenant: enablement, CLI application consent, baseline roles, and governance standards.
- Reusable per framework or hosting pattern: shared authentication, instrumentation wrappers, deployment configuration, telemetry conventions, and validation procedures.
- Per agent: blueprint and permissions, package and publication, instrumentation coverage, privacy review, end-to-end ingestion tests, and operational handover.

Observability usually drives the most variation. Auto-instrumentation for a supported framework reduces code changes, while custom runtimes, direct OTLP export, multiple model/tool providers, sub-agents, asynchronous jobs, and mixed OBO/S2S flows require more manual instrumentation and testing. Other sizing drivers include the number of environments, network egress constraints, permission-consent lead time, telemetry redaction requirements, and the number of distinct invocation, tool, and failure paths.

## Recommended Custom Agent Onboarding Flow

### 0. Enable the tenant and install the CLI

Agent 365 licensing and tenant enablement are separate gates: a licensed tenant is not automatically enabled. A Global Administrator must complete the tenant requirement setup before downstream registration and integration can proceed.

The Agent 365 CLI is a .NET global tool and requires .NET 8 or later on the workstation, even when the agent itself is implemented in Python or Node.js.

```bash
dotnet tool install --global Microsoft.Agents.A365.DevTools.Cli
a365 setup requirements
```

The requirements command creates the `Agent 365 CLI` enterprise application and obtains admin consent for the required Microsoft Graph scopes. Treat this as tenant bootstrap work rather than recurring per-agent development.

### 1. Classify the agent

Before implementation, record:

- Agent name and business purpose.
- Owner and business sponsor.
- Runtime and framework.
- Hosting location.
- Channels where users will access it.
- Data sources and tools it can read or modify.
- Authentication mode: delegated user access, application access, or both.
- Required Microsoft Graph permissions.
- Whether it needs Work IQ or other MCP tools.
- Whether it needs an AI teammate identity.
- Logging, retention, and compliance requirements.

### 2. Choose the identity model

For new agents, prefer an Agent 365 agent identity blueprint when Agent 365 governance capabilities are required. Existing Microsoft Entra application registrations can continue to work, but blueprint-based agents get stronger alignment with Agent 365 registration, Work IQ, governance, and security controls.

Common identity patterns:

- On-Behalf-Of (OBO): the agent acts with the signed-in user's delegated permissions. Prefer this for user-specific mail, files, calendar, Teams, and SharePoint access.
- Service-to-service (S2S): the agent acts as its own application identity with application permissions. Use only where background or autonomous processing is required.
- Mixed OBO and S2S: use when both interactive user-scoped access and autonomous tasks are required.
- AI teammate identity: the agent has its own Microsoft 365 user-like identity. Use only when the agent must appear as a participant with mailbox, Teams presence, directory metadata, and lifecycle management.

### 3. Register an agent blueprint

Install and configure the Agent 365 CLI, then create the agent blueprint.

For a regular agent:

```bash
a365 setup all --agent-name "claims-triage-agent"
```

For an agent already available in Teams or Copilot:

```bash
a365 setup all --m365
```

For an AI teammate:

```bash
a365 setup all --aiteammate
```

The setup process can create Azure infrastructure if needed, but deployment to Azure is optional when an agent is already hosted elsewhere. The setup registers the agent blueprint in Microsoft Entra, creates app registrations, configures permissions, and writes generated values to `a365.generated.config.json`.

Required roles and permissions depend on the task:

- Global Administrator can complete setup and OAuth consent.
- Agent ID Developer can complete most setup steps but must hand off consent URLs to a Global Administrator.
- Azure subscription permissions are required if the CLI creates Azure resources.

Blueprint setup is required for Agent 365 Register, Work IQ, and AI teammate capabilities. It is also the recommended path for new custom agents.

### 4. Declare required permissions

Grant only the Microsoft Graph and MCP permissions the agent needs.

Example custom Microsoft Graph permissions:

```bash
a365 setup permissions custom \
  --resource-app-id 00000003-0000-0000-c000-000000000000 \
  --scopes Mail.Read,Mail.Send,User.Read
```

Review permissions in Microsoft Entra and in the Microsoft 365 admin center before deployment. Application permissions can be broad and should be minimized.

### 5. Package and publish

Use the CLI to update the Microsoft 365 app manifest and create the upload package:

```bash
a365 publish
```

The publish command updates `manifest.json` with the agent blueprint ID, packages `manifest.json` and icons into `manifest.zip`, and prints upload instructions.

Admin upload flow:

1. Open Microsoft 365 admin center.
2. Go to Agents > All agents > Add agent.
3. Upload the `manifest.zip` package.
4. Validate name, icon, and host products.
5. Select users or groups who can install the agent.
6. Optionally select users or groups who receive the agent preinstalled.
7. Apply an existing policy template, custom policy, or default policy.
8. Review permissions.
9. Finish deployment.

After upload, allow time for the agent to appear in Microsoft 365 admin center and host experiences.

## Observability Requirements

Observability is required for meaningful governance. Without it, administrators may know an agent exists, but they cannot reliably inspect behavior, tool calls, model calls, errors, or outputs.

For new integrations, Microsoft recommends the Microsoft OpenTelemetry Distro. The older Agent 365 Observability SDK remains documented and continues to work, but direct OTel and SDK guidance should be evaluated against the latest Microsoft documentation before implementation.

### Auto-instrumentation path

Python packages documented for Agent 365 observability include:

```bash
pip install microsoft-agents-a365-observability-core
pip install microsoft-agents-a365-runtime
pip install microsoft-agents-a365-observability-extensions-langchain
```

Example LangChain-style configuration:

```python
from microsoft_agents_a365.observability.core.config import configure
from microsoft_agents_a365.observability.extensions.langchain import CustomLangChainInstrumentor


def token_resolver(agent_id: str, tenant_id: str) -> str | None:
    # Use MSAL, managed identity, or another approved token flow.
    # The token must include the Agent365.Observability.OtelWrite scope or role.
    return f"Bearer {get_agent365_observability_token(agent_id, tenant_id)}"


configure(
    service_name="claims-triage-agent",
    service_namespace="contoso.agents",
    token_resolver=token_resolver,
)

CustomLangChainInstrumentor()
```

Auto-instrumentation support varies by language and framework. Documented Python support includes Semantic Kernel, OpenAI Agents SDK, Microsoft Agent Framework, and LangChain.

### Manual instrumentation path

Manual instrumentation should capture at least:

- Agent invocation.
- LLM inference or chat call.
- Tool execution.
- Output messages.
- Exceptions.
- Tenant ID, agent ID, conversation ID, channel, caller, and endpoint details.

Store publishing validation requires `InvokeAgentScope`, `InferenceScope`, and `ExecuteToolScope` equivalents.

### Direct OpenTelemetry path

Use direct OTLP/HTTP only when an existing OpenTelemetry pipeline is already in place, the Agent 365 SDK cannot be used, or the implementation language is not supported by the SDK.

Endpoint pattern for S2S telemetry:

```text
POST https://agent365.svc.cloud.microsoft/observabilityService/tenants/{tenantId}/otlp/agents/{agentId}/traces?api-version=1
```

Endpoint pattern for delegated/OBO telemetry:

```text
POST https://agent365.svc.cloud.microsoft/observability/tenants/{tenantId}/otlp/agents/{agentId}/traces?api-version=1
```

Minimal Python exporter shape:

```python
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

exporter = OTLPSpanExporter(
    endpoint=(
        "https://agent365.svc.cloud.microsoft/"
        "observabilityService/tenants/{tenantId}/otlp/agents/{agentId}/traces?api-version=1"
    ),
    headers={"Authorization": f"Bearer {token}"},
)
```

Direct OTel requirements and caveats:

- The tenant must have a qualifying Microsoft 365 E7 or Microsoft Agent 365 license assigned to at least one user. SKU presence alone is not enough for ingestion.
- Tenant admin consent is required.
- The app or blueprint must have `Agent365.Observability.OtelWrite`.
- `invoke_agent` is required for runs to appear in Microsoft 365 admin center activity views.
- Supported operation names include `invoke_agent`, `chat`, `execute_tool`, and `output_messages`.
- All spans in a run should share a trace ID and conversation ID.
- Non-root spans must set `parentSpanId`.
- Always parse `partialSuccess`; `200 OK` is not sufficient proof that telemetry landed.
- Request payloads must stay within service limits.

## Tool Governance With Work IQ and MCP

Prefer governed Work IQ MCP servers or approved MCP servers for important Microsoft 365 and enterprise actions. Tool calls that bypass Agent 365 tooling and emit no telemetry create governance blind spots.

Work IQ MCP servers are currently documented as preview and require Microsoft 365 Copilot licensing. Availability can vary by region.

Common Work IQ and Agent 365 MCP capabilities include:

- Copilot
- Calendar
- Mail
- SharePoint
- OneDrive
- Teams
- User profile and org data
- Word
- Dataverse and Dynamics 365

### Add MCP servers to an agent

Discover and configure servers:

```bash
a365 develop list-available
a365 develop add-mcp-servers mcp_MailTools
a365 develop list-configured
```

The CLI writes `ToolingManifest.json`. This only configures the project; it does not grant production permissions.

Grant MCP permissions:

```bash
a365 setup permissions mcp
```

The permission step requires Global Administrator consent. The configured MCP servers are not usable in production until consent is complete.

### Integrate MCP tools in Python

Indicative OpenAI extension shape:

```python
from microsoft.agents.a365.tooling import McpToolServerConfigurationService
from microsoft.agents.a365.tooling.extensions.openai import mcp_tool_registration_service

config_service = McpToolServerConfigurationService()
tool_service = mcp_tool_registration_service.McpToolRegistrationService()

agent = await tool_service.add_tool_servers_to_agent(
    agent=agent,
    agentic_app_id=agentic_app_id,
    auth=auth,
    context=context,
)
```

For early development, use the mock tooling server:

```bash
a365 develop start-mock-tooling-server
```

Set `MCP_PLATFORM_ENDPOINT=http://localhost:5309` for local testing.

### Bring Your Own MCP server

BYO MCP server registration lets organizations route custom MCP servers through the Agent 365 tooling gateway for review, approval, blocking, and monitoring.

Supported authentication types include:

- NoAuth
- APIKey in header or query
- ExternalOAuth
- EntraOAuth

Example registration shape:

```bash
a365 develop-mcp register-external-mcp-server \
  --server-name "InternalDocsSearch" \
  --server-url "https://docs.contoso.com/api/mcp" \
  --publisher "Contoso" \
  --description "Documentation search MCP server" \
  --auth-type "NoAuth" \
  --tools "search_docs"
```

The admin review flow is:

1. Developer registers the remote MCP server.
2. AI Administrator or Global Administrator reviews it under Agents > Tools > Requests.
3. Admin approves or rejects the server.
4. Admin grants required Microsoft Entra permissions.
5. Security teams monitor invocations in Microsoft Defender advanced hunting.

BYO MCP server is preview. Supported client surfaces currently include Copilot Studio, VS Code, Claude Code, and GitHub Copilot CLI. Azure AI Foundry and Microsoft 365 Declarative Agents are not yet supported for this BYO MCP server flow.

## Admin Governance Surfaces

Primary admin surface:

```text
Microsoft 365 admin center > Agents
```

Key registry capabilities:

- View all available agents in the tenant.
- Filter by status, publisher type, channel, platform, and data source.
- Identify ownerless agents.
- Identify unmanaged agents.
- Review high-severity risks aggregated from Microsoft Entra, Microsoft Purview, and Microsoft Defender.
- Upload custom agent packages.
- Export agent inventory.
- Manage pinned agents.
- Use preview Microsoft Graph package APIs for inventory and details.

Agent detail panes can include tabs such as:

- Details
- Users
- Data & Tools
- Security
- Permissions
- Certification
- Activity
- Agent instances
- Connected agents
- Computer use

Not every tab appears for every agent. Tabs depend on agent type, platform, capabilities, and licensing.

Common lifecycle actions include:

- Install
- Uninstall
- Block
- Unblock
- Update in store
- Pin for users
- Delete, where supported
- Assign new owner, where supported

Agent availability and installation are separate controls:

- Published to: controls who can discover and install the agent.
- Installed to: controls who receives the agent preinstalled.

## Tenant-Level Settings

Use Agents > Settings to define tenant-level governance.

Settings include:

- Agent management rules.
- Allowed agent types.
- Security templates.
- Sharing controls.
- User access controls.

Allowed agent types can permit or restrict:

- Microsoft-built agents.
- Agents built by the organization.
- Agents built by external publishers.

Sharing controls currently apply only to Microsoft 365 Copilot Agent Builder agents.

Agent management rules currently support a limited set of bulk scenarios, such as installing Microsoft first-party agents and reassigning ownerless Agent Builder agents to the previous owner's manager.

## Roles and Permissions

Use least privilege.

Important roles:

- Global Administrator: full tenant-wide visibility and governance authority.
- AI Administrator: tenant-wide agent visibility and governance authority.
- Global Reader and AI Reader: read-only visibility with limited/no governance actions.
- Security Administrator and Security Reader: security visibility, not broad governance authority.
- Reports Reader and User Experience Success Manager: reporting/monitoring visibility.

Global Administrator should be limited to scenarios where AI Administrator or another lower-privilege role cannot complete the task.

## Security and Compliance Integration

Agent 365 integrates with the Microsoft security and compliance stack.

### Microsoft Entra

Use Microsoft Entra Agent ID and related controls for:

- Agent identities.
- Conditional Access.
- Identity Protection.
- Access reviews.
- Lifecycle governance.
- Sign-in and audit logs.
- Agent identity blueprints.

### Microsoft Purview

Use Microsoft Purview for:

- Data Security Posture Management.
- Auditing.
- Data classification.
- Sensitivity labels.
- Data Loss Prevention.
- Insider Risk Management.
- Communication Compliance.
- eDiscovery.
- Retention and data lifecycle management.
- Compliance Manager assessments.

When creating an Agent 365 agent instance, audit, sensitive data detection, and Compliance Manager AI assessments are enabled automatically. Other Purview capabilities often require including the agent instance in policies as if it were a user.

Important Purview caveats:

- Files must be explicitly shared with agent instances for access.
- Encrypted sensitivity labels must grant the agent VIEW and EXTRACT usage rights.
- Newly created content from Agent 365 does not automatically inherit source sensitivity labels.
- Some DLP block actions may not be visible to the agent itself, so owners must monitor policy impact.

### Microsoft Defender

Use Microsoft Defender for:

- Agent security posture.
- Threat detection.
- Prompt injection and tool misuse signals where supported.
- Incidents and alerts.
- Advanced hunting over agent and tool activity.

For MCP tool calls routed through the Agent 365 gateway, Defender advanced hunting can inspect tool invocation activity. For custom agents that call tools directly without telemetry, Defender and Agent 365 visibility will be limited.

## Agent Manifest and Agent Card Concepts

There is a manifest concept in Microsoft 365 and Agent 365 workflows.

### Microsoft 365 app manifest

The Microsoft 365 app manifest is a JSON file that describes how an app or agent integrates with Microsoft 365 products such as Copilot, Teams, Outlook, Word, and other hosts. It is packaged with icons and optional agent definitions in a ZIP app package.

Agent 365 publishing uses this package format. The `a365 publish` command updates `manifest.json` and creates `manifest.zip`.

### Declarative agent manifest

Declarative agents have their own manifest schema. The current documented schema is v1.7.

Common declarative agent manifest properties include:

- `version`
- `name`
- `description`
- `instructions`
- `capabilities`
- `conversation_starters`
- `actions`
- `behavior_overrides`
- `disclaimer`
- `sensitivity_label`
- `editorial_answers`
- `worker_agents`
- `user_overrides`

Example minimal declarative agent manifest:

```json
{
  "version": "v1.7",
  "name": "Repairs agent",
  "description": "Helps track tickets and repairs",
  "instructions": "Use approved repair and ticket sources to answer questions about open repair work."
}
```

### Tooling manifest

Agent 365 tooling uses `ToolingManifest.json` to list configured MCP servers, scopes, and audiences. The CLI creates this file when MCP servers are added.

### Agent card

Agent 365 registration is not based on a single universal agent card artifact. The general Agent 365 onboarding model is based on identity, blueprints, Microsoft 365 app packages, manifests, registry records, and telemetry.

There is an agent card concept in Microsoft Foundry Agent Service A2A preview. When incoming A2A is enabled on a Foundry agent, Foundry exposes an authenticated agent card endpoint such as:

```text
https://{account}.services.ai.azure.com/api/projects/{project}/agents/{agent}/endpoint/protocols/a2a/agentCard/v0.3
```

That agent card describes capabilities, skills, and metadata for A2A discovery. It is Foundry/A2A-specific, uses A2A protocol version 0.3, requires Microsoft Entra authentication, supports text modality only, and is not recommended for production workloads while in preview.

## Differences by Agent Type

### Copilot Studio and Agent Builder agents

Advantages:

- Automatic registry integration.
- Automatic telemetry for invocations and connector/tool usage.
- Built-in integration with Power Platform policy controls.
- Easier admin discovery and governance.
- Strong low-code support.

Tradeoffs:

- Less direct control over orchestration internals.
- Some lifecycle actions may need Power Platform environment administration.

### Microsoft Foundry agents

Advantages:

- Managed agent runtime, model/tool integration, tracing, evaluations, and publishing.
- Supports prompt agents, workflow agents, and hosted agents.
- Hosted agents can use custom frameworks such as LangGraph or Agent Framework.
- Can publish to Microsoft 365 Copilot, Teams, and Entra Agent Registry.

Tradeoffs:

- Some capabilities are preview, including hosted agents and A2A endpoints.
- Microsoft 365 integration may use a bot or proxy layer depending on publishing approach.
- Foundry governance is not identical to Agent 365 governance; align both control planes deliberately.

### Self-hosted custom agents

Advantages:

- Maximum control over model provider, orchestration, tools, deployment, and architecture.
- Can run on Azure, AWS, GCP, on-premises, or another endpoint.

Tradeoffs:

- Development teams own identity design, registration, consent, telemetry, tool governance, deployment, scaling, and runtime security.
- Admin center Data & Tools metadata may be incomplete unless the platform or integration reports it.
- Uninstrumented tool calls and sub-agent calls create governance blind spots.

## Known Limitations and Caveats

- Registry sync is preview and supports a limited set of external platforms.
- Work IQ MCP is preview and requires Microsoft 365 Copilot licensing.
- BYO MCP server is preview and has limited supported client surfaces.
- Incoming A2A for Foundry agents is preview, supports A2A protocol version 0.3 only, supports text modality only, and is not recommended for production workloads.
- AI teammate is available only to Frontier program participants and requires additional identity, licensing, mailbox, Teams, and lifecycle considerations.
- Data & Tools metadata in Microsoft 365 admin center is not available for all agent types.
- Activity metrics are currently supported for a subset of agent types, including Microsoft 365 Copilot Agent Builder, SharePoint, and Microsoft 365 Agents Toolkit agents.
- Risk columns and some Security/Activity tab details require appropriate licensing.
- Microsoft Graph Package Management APIs for agent registry and details are preview/beta and should not be treated as stable production automation contracts.
- Package Management API access requires a Microsoft Agent 365 license and currently supports delegated work or school permissions, not application permissions, for documented list/detail operations.
- Admin-pinned agents are limited to reserved slots and can take hours to appear for users.
- Some Microsoft 365 Copilot declarative agent capabilities require Microsoft 365 Copilot licensing or metered usage.
- Embedded knowledge behavior differs between Agent Builder service-managed embedded content and declarative agent manifest embedded knowledge documentation. Validate file size and feature availability for the authoring surface being used.

## Production Readiness Checklist

Use this checklist before broad deployment.

- Agent has a named owner and sponsor.
- Agent purpose and expected behavior are documented.
- Agent identity model is approved.
- Agent blueprint or Entra app registration exists.
- Required Microsoft Graph permissions are minimized and approved.
- Required MCP or Work IQ permissions are minimized and approved.
- Observability is implemented and tested.
- Tool calls, LLM calls, sub-agent calls, and outputs are traceable.
- Telemetry appears in Microsoft 365 admin center Activity where supported.
- Telemetry is visible in Defender/Purview where required.
- Agent package upload succeeds.
- Agent is first published to a small pilot group.
- Policy template or custom policy is applied.
- DLP and sensitivity label behavior is tested.
- Application permissions are reviewed for least privilege.
- Error and exception monitoring is configured.
- Ownerless-agent process is defined.
- Blocking, uninstall, and retirement process is tested.
- Compliance retention and eDiscovery requirements are documented.
- Third-party or non-Microsoft data handling is reviewed.
- Preview features are explicitly accepted by the risk owner before production use.

## Operating Principles

- Register every production agent. Unregistered agents are shadow risk.
- Prefer blueprint-based onboarding for new custom agents.
- Prefer OBO for user-scoped data access.
- Use S2S/application permissions sparingly and document why they are needed.
- Instrument before broad deployment.
- Route high-risk tool calls through Work IQ or approved MCP servers where possible.
- Treat direct, unobserved API calls as governance exceptions.
- Keep agent permissions, tool access, data sources, and model behavior reviewable by admins.
- Start with narrow deployment groups and expand after validation.
- Revalidate preview features and licensing before production rollout.

## Primary Microsoft References

- https://learn.microsoft.com/en-us/microsoft-agent-365/overview
- https://learn.microsoft.com/en-us/microsoft-agent-365/connect-existing-agents
- https://learn.microsoft.com/en-us/microsoft-agent-365/developer/get-started
- https://learn.microsoft.com/en-us/microsoft-agent-365/developer/reference/cli/
- https://learn.microsoft.com/en-us/microsoft-agent-365/developer/registration
- https://learn.microsoft.com/en-us/microsoft-agent-365/developer/publish
- https://learn.microsoft.com/en-us/microsoft-agent-365/developer/observability
- https://learn.microsoft.com/en-us/microsoft-agent-365/developer/direct-open-telemetry-integration
- https://learn.microsoft.com/en-us/microsoft-agent-365/developer/tooling
- https://learn.microsoft.com/en-us/microsoft-agent-365/tooling-servers-overview
- https://learn.microsoft.com/en-us/microsoft-365/admin/manage/agent-registry
- https://learn.microsoft.com/en-us/microsoft-365/admin/manage/agent-details
- https://learn.microsoft.com/en-us/microsoft-365/admin/manage/agent-settings
- https://learn.microsoft.com/en-us/microsoft-365/admin/manage/agent-roles-perms
- https://learn.microsoft.com/en-us/entra/agent-id/what-is-microsoft-entra-agent-id
- https://learn.microsoft.com/en-us/purview/ai-agent-365
- https://learn.microsoft.com/en-us/microsoft-365/extensibility/schema
- https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/declarative-agent-manifest-1.7
- https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/enable-agent-to-agent-endpoint
