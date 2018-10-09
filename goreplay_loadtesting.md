# Purpose
- Introduction to GoReplay
- Writing Middleware tailored to your needs

# Goal
- Gauge performance of would-be servers with realistic traffic
- Tune server specifications to avoid over-provisioning, (or under)

# Problem Background
## Loadtesting:
- Manually:
  - Can diversify requests
  - Slow, unrealistic throughput

- Scripted:
  - e.g. Apache Bench, online tools
  - Variable throughput
  - Typically limited in diversity

- GoReplay:
  - Requests are as diverse as the captured traffic
  - Traffic can be captured and replayed at customisable throughput (n/a for time-sensitive applications)
  - Traffic can be rate-limited

# Solution
## GoReplay installation
## Commands to run
## HAProxy Config workaround
## Middleware
## Monitoring
## Side Effects

# Conclusion
