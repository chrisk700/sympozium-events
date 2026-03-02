# Calendar

Fires on a cron schedule or fixed interval. Useful for periodic tasks, digests, and heartbeat checks.

## Configuration

```yaml
sources:
  - type: calendar
    name: morning-trigger
    schedule: "0 9 * * 1-5"   # cron — Mon–Fri at 09:00
    timezone: Europe/London
```

Or use an interval instead of a cron expression:

```yaml
sources:
  - type: calendar
    name: hourly-check
    interval: 1h
```

| Field | Required | Default | Description |
|---|---|---|---|
| `name` | yes | — | Source name. |
| `schedule` | no* | — | Cron expression (standard 5-field). Takes precedence over `interval` when both are set. |
| `interval` | no* | `60s` | Fixed interval. Accepts Go duration strings: `30s`, `5m`, `1h`. |
| `timezone` | no | UTC | Timezone for `schedule` interpretation. IANA tz name (e.g. `America/New_York`). |

*At least one of `schedule` or `interval` should be set. If neither is set, the source defaults to a 60-second interval.

## taskTemplate

The event time is available at `.Input.eventTime`:

```
Scheduled event fired at {{ .Input.eventTime }}
```

Override per target:

```yaml
targets:
  - instanceRef: sre-watchdog
    taskTemplate: |
      Daily SRE digest at {{ .Input.eventTime }}.
      Review overnight alerts and summarise.
```

## Example — weekday morning digest

```yaml
routes:
  - name: daily-digest
    sources:
      - type: calendar
        name: morning-trigger
        schedule: "0 9 * * 1-5"
        timezone: Europe/London
    targets:
      - instanceRef: sre-watchdog
        taskTemplate: |
          Daily digest at {{ .Input.eventTime }}.
          Check all systems and produce a summary.
```
