# API Specification

## Command Gateway

The desktop frontend should communicate with the Rust backend through a typed command gateway.

### Example Commands

- `send_message`
- `list_conversations`
- `get_task_status`
- `install_plugin`
- `configure_provider`

## Response Shape

Responses should be strongly typed, include success/error indicators, and surface execution metadata where needed.
