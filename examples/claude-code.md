# Claude Code Integration

Configure Claude Code to send telemetry to your LGTM stack.

## Configuration

On your machine running Claude Code:

```bash
# Enable telemetry
export CLAUDE_CODE_ENABLE_TELEMETRY=1

# Configure OpenTelemetry exporters
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp

# Point to your LGTM stack
export OTEL_EXPORTER_OTLP_ENDPOINT=http://YOUR_SERVER_IP:4317

# Add identifying attributes
export OTEL_RESOURCE_ATTRIBUTES="service.name=claude-code,host.name=$(hostname),user.name=$(whoami)"

# Optional: Include user prompts (privacy concern!)
# export OTEL_LOG_USER_PROMPTS=1
```

## Multiple Claude Code Instances

Different users/machines can be identified:

```bash
# Developer laptop
export OTEL_RESOURCE_ATTRIBUTES="service.name=claude-code,host.name=laptop,user.name=alice,team=frontend"

# CI/CD server
export OTEL_RESOURCE_ATTRIBUTES="service.name=claude-code,host.name=ci-server,environment=ci"
```

## Metrics Collected

Claude Code sends these metrics:
- `claude_code.session.count` - Number of sessions
- `claude_code.lines_of_code.count` - Lines modified
- `claude_code.token.usage` - Tokens consumed
- `claude_code.cost.usage` - Session costs
- `claude_code.pull_request.count` - PRs created
- `claude_code.commit.count` - Commits made
- `claude_code.active_time.total` - Active development time

## Sample Grafana Queries

```promql
# Sessions per user
sum by (user_name) (rate(claude_code_session_count[5m]))

# Token usage by team
sum by (team) (claude_code_token_usage)

# Cost per environment
sum by (environment) (claude_code_cost_usage)
```

## Dashboard Template

Create `dashboards/claude-code.json`:

```json
{
  "dashboard": {
    "title": "Claude Code Metrics",
    "panels": [
      {
        "title": "Sessions by User",
        "targets": [{
          "expr": "sum by (user_name) (rate(claude_code_session_count[5m]))"
        }]
      },
      {
        "title": "Lines of Code Modified",
        "targets": [{
          "expr": "sum(claude_code_lines_of_code_count)"
        }]
      }
    ]
  }
}
```