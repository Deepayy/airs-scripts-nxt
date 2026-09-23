# AI Gateway Demo Script

A self-contained script that exercises the Prisma AIRS AI Gateway end to end:
LLM calls, runtime security guardrails, MCP tool access, and model routing.
Each section prints what it is doing, the gateway's verdict, latency, and
which dashboard to look at afterwards. No API keys are stored in the script;
everything comes from environment variables.

## Requirements

- Python 3.9+ with the `requests` package (`pip install requests`)
- A workspace API key for your AI Gateway tenant
- The integrations referenced below provisioned to that workspace

## Setup

Create a file named `.env` next to the script (no spaces around `=`, no
quotes) and fill in the values for your tenant:

```
# Required
GW_URL=https://aigw.portkey.ai/v1
GW_KEY=<your workspace API key>

# Optional: which provider integration serves sections 1 and 2
# (defaults to @anthropic-main; use the @slug of any LLM integration
# provisioned to your workspace)
DEMO_PROVIDER=@shared-ai-nonprod-gemini

# Optional: model name for sections 1 and 2 (defaults to claude-haiku-4-5;
# must be a model your integration exposes, exactly as the Model Catalog
# names it)
DEMO_MODEL=gemini-2.0-flash

# Optional: guardrail config id, ONLY if a guardrail config is not already
# set as the default on your API key. If your key has a default config with
# AIRS guardrails attached, leave this unset.
DEMO_CFG=

# Optional: MCP section. The DeepWiki URL is the gateway-issued URL from
# your MCP Registry entry (defaults to .../deepwiki/mcp on the shared MCP
# host). Set the GitHub one only if you have a PAT-authenticated GitHub
# MCP server registered; leave unset to skip it.
MCP_URL_DEEPWIKI=https://mcp-aigw.portkey.ai/deepwiki/mcp
MCP_URL_GITHUB=

# Optional: model routing section. Set to the config id of the tier
# routing config described below; leave unset to skip the section.
DEMO_ROUTE_CFG=
```

Load the variables and run:

```
set -a && source .env && set +a
python3 aigw_demo.py
```

Run a single section with `python3 aigw_demo.py llm` (sections: `llm`,
`guard`, `mcp`, `route`), or rehearse without pauses using
`NO_PAUSE=1 python3 aigw_demo.py`.

## What each section needs

**1. LLM traffic** needs only GW_URL, GW_KEY, and an LLM integration
provisioned to your workspace. Three benign calls with metadata attached;
afterwards, open Logs and filter to your workspace to see them with tokens,
cost and latency.

**2. Guardrails** needs an AIRS guardrail attached to your key (as the
key's default config, or via DEMO_CFG). It sends one benign control and
seven unsafe prompts: prompt injection, a jailbreak attempt, payment and
identity data, cloud credentials, toxic content, a malicious URL (Google's
harmless test domain), and an output-leg case. Expect HTTP 446 on the
unsafe ones if your profile's actions are set to Block; HTTP 200 with a
logged verdict means the profile is in monitor mode, which is also a valid
demo. All test data is fake. Verdicts appear both in gateway Logs and in
Strata Cloud Manager with a scan id.

**3. MCP** needs at least one MCP server registered in your MCP Registry
with your workspace enabled under its Access Control. DeepWiki
(https://mcp.deepwiki.com/mcp, no upstream auth) is the easiest first
server. The section initialises, lists tools, and makes several tool calls;
a good live moment is toggling one tool off for your workspace in the
registry and re-running to watch it disappear from the list.

**4. Model routing** shows tier-based routing: the script classifies each
prompt by length, tags it in metadata, and the gateway's config decides
which model serves each tier. Create a config like the one below in your
workspace (Configs > Create), adjust the provider slugs and models to
integrations you have, and put its id in DEMO_ROUTE_CFG:

```json
{
  "strategy": {
    "mode": "conditional",
    "conditions": [
      {"query": {"metadata.tier": {"$eq": "short"}}, "then": "fast"},
      {"query": {"metadata.tier": {"$eq": "long"}}, "then": "detailed"}
    ],
    "default": "fast"
  },
  "targets": [
    {"name": "fast", "provider": "@<your-provider-slug>",
     "override_params": {"model": "<small-fast-model>"}},
    {"name": "detailed", "provider": "@<your-provider-slug-2>",
     "override_params": {"model": "<larger-model>"}}
  ]
}
```

The two targets can use the same provider with different models, or two
different providers. The point of the demo: the calling code never names a
model, and changing the tier-to-model mapping is a config edit that takes
effect for every app immediately.

## Reading the output

- ALLOWED 200: request served; the reply snippet and token count print
- BLOCKED 446: the AIRS guardrail denied it; the gateway's verdict body
  prints so you can see the detection category
- LIMITED 429: a rate limit or budget on your key fired
- Other codes print the gateway's error body; the most common causes are a
  key from a different workspace than the config or guardrail it references,
  an integration not provisioned to your workspace, or a model name that
  does not match the Model Catalog exactly

## Troubleshooting

- 401: wrong key, or the key is not a workspace API key
- 400 "guardrails are not valid": the config or guardrail lives in a
  different workspace than the key; create them in the same workspace
- 404 model errors: copy the model name verbatim from your Model Catalog
- MCP initialize failing while LLM calls work: check the registry entry's
  target URL (for DeepWiki it must be https://mcp.deepwiki.com/mcp) and
  that your workspace is enabled under the server's Access Control
- Section 2 prompts returning 200: your AIRS profile actions are set to
  allow/monitor rather than block, or that detector is disabled in the
  profile

Every request the script sends is tagged with metadata
(user "demo", app "live-demo") so you can filter for the demo traffic in
Logs and Analytics and remove any budgets or limits you set afterwards.
