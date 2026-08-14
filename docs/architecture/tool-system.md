# Tool System

## Overview

Tools provide the agent with actions it can execute against the desktop or external systems.

## Built-in Tool Categories

- Computer control tools
- Filesystem tools
- Browser tools
- System tools
- Notification tools

## Tool Execution Contract

Each tool should have a typed input schema, an approval policy, and a structured result payload. Execution should be observable so the UI can render the action chain.

## Safety Rules

- Require capability checks before privileged actions.
- Record all tool invocations in the audit log.
- Fail closed when permission or context is missing.
