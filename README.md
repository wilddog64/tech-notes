# Notes Repository

This repository contains technical notes categorized by topic. Each section provides in-depth guidance, troubleshooting steps, and configuration tips for working with specific DevOps tools.

## 📁 AI

- [gpt-oss introduction](ai/gpt-oss/gpt-oss.md)
- [gpt-oss - setup and running on m4 air](ai/gpt-oss/m4-setup.md)

## 📁 Apple/Mac OS

- [Mac Uinversal Control Troubleshoot guide](apple/mac-universal-control.md)

## 📁 Jenkins

Automation and pipeline-related notes for Jenkins, including memory tuning, validation, and migration strategies.

- [Memory and CPU limits for a Jenkins container](jenkins/jenkins-memory.md)
- [Pipeline Migration](jenkins/pipeline-migration.md)
- [Use Jenkins replay to validate pipeline migration](jenkins/pipeline-replay.md)
curl http://localhost:11434/api/generate -d '{"model":"gpt-oss:20b","prompt":"ready","options":{"num_ctx":4096},"keep_alive":"2h"}' 2>&1 &
curl http://localhost:11434/api/generate -d '{"model":"gpt-oss:20b","prompt":"ready","options":{"num_ctx":4096},"keep_alive":"2h"}' 2>&1 &

## 📁 Kubernetes

Guides and debugging references for Kubernetes environments.

- [Kubernetes PVC/PV + SMB CSI Troubleshooting Guide](kubernetes/pvc-pv-smb-troubleshooting-guide.md)
