# Testing Strategy

## Test Layers

- Unit tests for business logic and provider adapters
- Integration tests for command gateway behavior
- End-to-end tests for desktop automation workflows
- Security and permission tests for privileged actions

## Quality Gates

- All permissioned actions require explicit coverage.
- Errors should be deterministic and observable.
- Regression tests should lock down the command surface.
