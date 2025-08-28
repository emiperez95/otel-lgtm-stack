# Adding Custom Dashboards

To add custom Grafana dashboards to this stack:

## Method 1: Add Dashboard Files (Recommended)

1. Export or create your dashboard JSON file
2. Place it in the `../../dashboards/` directory
3. Restart Grafana: `docker-compose restart grafana`
4. Your dashboard will be automatically loaded

Example:
```bash
cp my-dashboard.json ../../dashboards/
docker-compose restart grafana
```

## Method 2: Create in Grafana UI

1. Open Grafana at http://localhost:8439
2. Create or import your dashboard
3. Save it to the "General" folder
4. To persist it, export the JSON and save to `../../dashboards/`

## Dashboard Naming Convention

Suggested naming pattern:
- `service-name-dashboard.json` - For service-specific dashboards
- `team-name-overview.json` - For team overviews
- `environment-metrics.json` - For environment-specific metrics

## Example Dashboard Structure

```json
{
  "dashboard": {
    "title": "My Service Metrics",
    "panels": [...],
    "tags": ["service", "production"]
  }
}
```

## Pre-built Dashboards

You can find community dashboards at:
- https://grafana.com/grafana/dashboards/
- Search for "OpenTelemetry" or your specific service

## Tips

- Use variables for dynamic filtering (service, environment, etc.)
- Set appropriate refresh intervals based on data frequency
- Use folders to organize dashboards by team or service
- Add tags for easier searching