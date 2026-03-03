# Notification policy defaults

**Use a policy like this:**

## Warning channel

### Send:
- severity=warning
- severity=info
- datasource no-data
- datasource error

### Destination:
- Slack
- Teams
- email

## Critical channel

### Send:
- severity=critical

### Destination:
- PagerDuty
- Opsgenie
- phone paging

## Grouping keys

### Group by:
- alertname
- cluster
- environment
- service

### Do not group by:
- pod
- instance
- container ID
- request path

