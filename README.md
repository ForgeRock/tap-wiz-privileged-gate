# Wiz Privileged Gate Node

A custom authentication node for PingAM that gates authentication on the user's Wiz risk posture — blocking access for users with open critical issues and forcing step-up authentication for users with open high-severity issues.

## Overview

During authentication, the node resolves the user's identity from shared state, authenticates to the Wiz API with client credentials, and counts the user's open issues (`OPEN` / `IN_PROGRESS`) with severity `CRITICAL` or `HIGH`. It writes the counts to shared state and applies the gate decision with stricter thresholds than a normal login flow would use.

Use this node to:

- Block authentication entirely when the user has open `CRITICAL` Wiz issues (e.g. a compromised or misconfigured cloud identity)
- Force step-up authentication (MFA, re-auth) when the user has open `HIGH` issues
- Keep privileged-access journeys aligned with cloud security posture in real time

### Gate decision

| Condition | Outcome |
|---|---|
| `critical count >= block threshold` | `block` |
| `high count >= step-up threshold` | `stepUp` |
| Otherwise | `allow` |

Critical issues take precedence: a user with both critical and high issues is blocked, not stepped up. Both comparisons use `>=` semantics, and both thresholds are configurable.

## Outcomes

| Outcome | Description |
|---|---|
| `allow` | No threshold met — continue the journey |
| `stepUp` | High-severity threshold met — route to step-up authentication |
| `block` | Critical-severity threshold met — deny access |
| `error` | The user id was missing, the Wiz configuration was incomplete, or the API call failed |

## Configuration

| Option | Type | Required | Default | Description |
|---|---|---|---|---|
| Auth URL | String | No | `` | Wiz OAuth 2.0 token endpoint. |
| API URL | String | Yes | `` | Tenant-specific Wiz GraphQL endpoint (varies by tenant region — copy from your Wiz tenant settings). |
| Client ID | String | Yes | *(empty)* | Wiz service-account client ID. |
| Client Secret | String | Yes | *(empty)* | Wiz service-account client secret. |
| Block On Critical | Number | No | `1` | Open critical issues at or above this value → `block`. |
| Step Up On High | Number | No | `1` | Open high issues at or above this value → `stepUp`. |

A fresh OAuth token (`grant_type=client_credentials`, `audience=wiz-api`) is obtained per node invocation; there is no token caching.

## Shared state

### Reads

| Key | Description |
|---|---|
| `mail` | User id used for the Wiz search — preferred |
| `username` | Fallback user id when `mail` is absent |

### Writes

| Key | Type | When | Description |
|---|---|---|---|
| `wizCriticalCount` | Number | Success | Open critical-severity issues |
| `wizHighCount` | Number | Success | Open high-severity issues |

Nothing is written on the `error` outcome. If this node follows the [Wiz Risk Enrichment](https://github.com/ForgeRock/tap-wiz-risk-enrichment) node in the same journey, the two counts are overwritten with the same values (both nodes run the same query).

## Error handling

| Condition | Behavior |
|---|---|
| Neither `mail` nor `username` in shared state | `error` outcome, no outbound call |
| Blank auth/API URL, client ID, or secret | `error` outcome (validated at token-request time) |
| Token or GraphQL HTTP failure | `error` outcome |
| GraphQL response contains `errors` | `error` outcome |
| Any other exception | `error` outcome |

On every error path the node logs the cause and takes the `error` outcome without writing shared state or a failure-node message.

**Fail-safe guidance:** for a gate like this, wire the `error` outcome to `Failure` (fail closed) if Wiz unavailability must never silently grant access, or to `Success`/step-up if availability matters more. This is journey wiring, not node behavior.

## Journey example

```
Start → [Username/Password collection] → [Data Store Decision] → [Wiz Privileged Gate]
                                                                    ├─ allow  → Success
                                                                    ├─ stepUp → [MFA / re-authentication]
                                                                    ├─ block  → Failure
                                                                    └─ error  → Failure (fail closed recommended)
```

## Wiz service account

This node requires a Wiz service account with API access and the `read:issues` scope (or equivalent role allowing issue search). Create one in your Wiz tenant under **Settings → Service Accounts**, then copy the **API URL**  and the client credentials into the node configuration.

## Installation

1. Obtain the node jar (`wiz-gti-marketplace-nodes-<version>.jar` — this jar also contains the companion nodes; see Related nodes).
2. Copy it into `webapps/openam/WEB-INF/lib/` (PingAM).
3. Restart the AM web application container.
4. The node appears in the journey designer as **Wiz privileged-access gate for the authenticated user**.

## Related nodes

- [Wiz Risk Enrichment](https://github.com/ForgeRock/tap-wiz-risk-enrichment)
- [Wiz Cloud Identity Check](https://github.com/ForgeRock/tap-wiz-cloud-identity-check)
- [GTI IP Reputation](https://github.com/ForgeRock/tap-gti-ip-reputation)

## Support

This integration is provided as a sample journey node. Review with your Ping Identity team before production use. For general PingAM/PingAIC node development, see the Ping Identity documentation on custom authentication nodes.
