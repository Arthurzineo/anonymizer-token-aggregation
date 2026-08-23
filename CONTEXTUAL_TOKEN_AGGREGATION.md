# Contextual token aggregation integration

## Why this integration exists

The Leakspok analyzer can optionally aggregate compatible adjacent tokens before
matching PHONE, CPF, CREDIT_CARD, and EMAIL rules. This detects formatted values such as
`+55 54 99912 0654`, `5 5 5 4 9 9 9 1 2 0 6 5 4`,
`529 982 247 25`, `5200 1000 0000 2803`, and `test @ example . com`, while
retaining the original input span for anonymization.

Anonymizer exposes that capability as an application-level feature flag. It is
off by default so existing deployments retain their current behavior and cost
until operators explicitly enable it.

## Changes

- Added `PRIVACY_CONTEXTUAL_DETECTION_ENABLED` to environment configuration.
- Propagated the setting into Leakspok's
  `RunnerOptions.ContextualDetection.Enabled` when the server creates analyzers.
- Documented the setting in the configuration reference.
- Added configuration and HTTP-level tests for disabled and enabled behavior.
- Added no new runtime dependency and made no change to request payloads or rule
  definitions.

## Enabling the feature

```sh
export PRIVACY_CONTEXTUAL_DETECTION_ENABLED=true
```

Restart the service after changing the environment. When enabled, contextual
aggregation remains bounded by Leakspok's token, byte, digit, separator, and
entity limits. Rule matching, exceptions, cache behavior, masking, and redaction
continue to use the existing implementation.

## Technical architecture changes

### Previous flow

Anonymizer loaded privacy configuration from environment variables, created the
Leakspok runner options in the server bootstrap, and reused the resulting
analyzer from its HTTP handlers. There was no application-level switch for
contextual candidate construction.

```text
environment -> Config -> server bootstrap -> Leakspok analyzer -> HTTP handler
```

### Updated flow

The same dependency path is retained. A new boolean is decoded with the other
privacy environment settings and passed by the server bootstrap into
`RunnerOptions.ContextualDetection.Enabled`. Handlers and request contracts do
not parse or override the option; they continue to use the configured analyzer.

```text
PRIVACY_CONTEXTUAL_DETECTION_ENABLED
              -> Config.Privacy
              -> server bootstrap
              -> Leakspok RunnerOptions
              -> shared analyzer
              -> HTTP handler
```

This keeps the policy deployment-wide, prevents request-controlled activation,
and preserves the existing service and handler boundaries. The default value is
`false`, so deployments must opt in explicitly.

### New files

- `CONTEXTUAL_TOKEN_AGGREGATION.md`: integration rationale, configuration,
  architecture, file inventory, and verification instructions.

### Modified original files

- `pkg/config/env.go`: adds and decodes
  `PRIVACY_CONTEXTUAL_DETECTION_ENABLED`, with a default of `false`.
- `pkg/config/env_test.go`: verifies both the default and environment-driven
  configuration behavior.
- `pkg/server/app.go`: maps the application setting into Leakspok's contextual
  runner option when constructing analyzers.
- `pkg/server/app_test.go`: updates server construction fixtures and checks the
  new configuration path.
- `internal/handler/handler_test.go`: adds an HTTP-level case proving that a
  fragmented value is detected when the feature is enabled.
- `docs/content/configuration.md`: documents the new environment variable,
  default, and purpose.
- `vendor/github.com/Prosus-Cyber-Xchange/leakspok/analyzer/byte_analyzer.go`,
  `factory.go`, `contextual_detection.go`, and `pattern/regex.go`: synchronized
  production snapshot so Docker and `-mod=vendor` builds use the same private
  Leakspok implementation as the workspace.

No new HTTP field, endpoint, rule schema, service layer, or runtime dependency
was introduced.

The vendored snapshot is appropriate for these paired private repositories. An
upstream release should instead tag Leakspok first, update the Anonymizer module
version, and regenerate vendor with the normal `task vendor` workflow.

### Bounded candidate examples

Candidate construction occurs inside Leakspok; Anonymizer only enables it for
the shared analyzer. The following examples illustrate what the integration can
detect when `PRIVACY_CONTEXTUAL_DETECTION_ENABLED=true`:

| Request text fragments | Canonical value evaluated by Leakspok | Outcome |
|---|---|---|
| `+55` `54` `99912` `0654` | `+5554999120654` | PHONE rule can redact the complete original span |
| `5` `5` `5` `4` `9` `9` `9` `1` `2` `0` `6` `5` `4` | `5554999120654` | supported within the 17-token and 16-digit numeric bounds |
| `529` `982` `247` `25` | `52998224725` | CPF rule accepts it only with a valid checksum |
| `5200` `1000` `0000` `2803` | `5200100000002803` | CREDIT_CARD rule requires a supported prefix and valid Luhn checksum |
| `test` `@` `example` `.` `com` | `test@example.com` | supported at the 5-token email bound |

Newlines, intervening words, and unsupported punctuation stop aggregation. The
scanner never extends numeric candidates beyond 16 digits, 17 tokens, or 64
bytes, nor email candidates beyond 5 tokens or 254 bytes. An over-limit chain is
rejected as a whole, so a shorter prefix is not anonymized while leaving a tail
visible. The canonical value is used only for validation; anonymization is
applied to the original request byte span, including its spaces and formatting.

PHONE continues to use Leakspok's permissive legacy matcher. Consequently,
unrelated numeric groups such as `Sala 101 202 303` may be classified as PHONE
when contextual detection is enabled. This known false-positive trade-off is
documented and is one reason the feature remains opt-in. A complete 16-digit
chain is isolated to CREDIT_CARD evaluation and additionally requires Luhn, so
it is not partially classified as PHONE.

Valid common dates are excluded before PHONE evaluation: Brazilian
`DD-MM-YYYY`, United States `MM-DD-YYYY`, and ISO `YYYY-MM-DD`, using `-`, `.`,
or `/`, with optional `HH`, `HHMM`, `HH:MM`, or `HH:MM:SS`. Calendar days and
leap years are validated. This is a bounded byte check over at most 64 bytes,
not a regex or natural-language parser. Invalid dates remain eligible for normal
rules, while a real phone deliberately formatted exactly as a valid date will
be treated as a date.

## Verification

The repository's existing configuration, handler, server, cache, and end-to-end
tests were retained, and integration-specific cases were added. Run the complete
suite with Docker available for testcontainers:

```sh
go test -race -count=1 ./...
```
