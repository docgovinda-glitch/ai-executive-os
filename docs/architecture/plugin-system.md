# Plugin System

## Overview

The plugin system uses the Model Context Protocol (MCP) to let third-party servers register tools, resources, and prompts.

## Plugin Lifecycle

1. Discovery
2. Startup and handshake
3. Registration
4. Tool exposure
5. Runtime monitoring
6. Shutdown and cleanup

## Security Boundaries

Plugins run in isolated processes and are only able to perform actions the user has granted. Tool permissions must be auditable and upgradeable by policy.
