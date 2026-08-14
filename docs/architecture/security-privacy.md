# Security & Privacy

## Security Objectives

- Keep data local by default.
- Use explicit user consent for privileged actions.
- Store secrets in the platform keychain.
- Preserve an audit trail for policy-sensitive operations.

## Privacy Model

No telemetry is required for core functionality. The application should avoid transmitting user content unless the user explicitly chooses a remote provider.

## Permission Model

Capabilities should be checked at the command gateway and again at the tool boundary whenever a privileged operation is attempted.
