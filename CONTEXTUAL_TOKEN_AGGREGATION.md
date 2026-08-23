# Contextual token aggregation integration

## Why this integration exists

The Leakspok analyzer can optionally aggregate compatible adjacent tokens before
matching PHONE, CPF, and EMAIL rules. This detects formatted values such as
`+55 54 99912 0654`, `5 5 5 4 9 9 9 1 2 0 6 5 4`,
`529 982 247 25`, and `test @ example . com`, while retaining the original input
span for anonymization.

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

## Verification

The repository's existing configuration, handler, server, cache, and end-to-end
tests were retained, and integration-specific cases were added. Run the complete
suite with Docker available for testcontainers:

```sh
go test -race -count=1 ./...
```
