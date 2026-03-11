---
title: New Relic Agent (Docker Compose) not showing container metrics
author: Jackson Hughes
pubDatetime: 2026-03-11T22:25:00Z
slug: newrelic-agent-docker-compose-no-metrics
featured: false
draft: false
tags:
  - newrelic-agent
  - newrelic-infra
  - docker
  - docker-compose
description: Fix the New Relic Infrastructure Agent not reporting container metrics when installed with Docker Compose.
timezone: "Europe/London"
---

## The Problem

I recently set up the New Relic Agent using Docker Compose. I could see my containers in the New Relic UI, but all of the container metrics were reporting zero values.

## The Cause

The guided install process and [New Relic's Docker Compose guide](https://docs.newrelic.com/docs/infrastructure/infrastructure-agent/linux-installation/infra-agent-as-container/#basic-compose) do not grant the agent access to the host's cgroup namespace.

On systems using **cgroup v2**, the agent reads container metrics from `/sys/fs/cgroup`. When the container runs in its default (private) cgroup namespace, it can only see its own cgroup — even with `privileged: true` — and not the rest of the host's container hierarchy. Therefore, the agent cannot read metrics from other containers on the host.

Interestingly, the `docker run` examples in the documentation do include the required flag: `--cgroupns=host`.

However, the Docker Compose examples omit the equivalent setting.

## The Solution

Therefore, the fix is to set the cgroup namespace to `host` for the service. The equivalent of `--cgroupns=host` in Docker Compose parlance is `cgroup: host`:

```yaml
# [!code --:1]
version: "3"
services:
  agent:
    container_name: newrelic-infra
    image: newrelic/infrastructure:latest
    cap_add:
      - SYS_PTRACE
    network_mode: host
    pid: host
    privileged: true
    # [!code ++:1]
    cgroup: host
    volumes:
      - "/:/host:ro"
      - "/var/run/docker.sock:/var/run/docker.sock"
    environment:
      NRIA_LICENSE_KEY: "YOUR_LICENSE_KEY"
    restart: unless-stopped
```

You can find more details in the [Compose reference](https://docs.docker.com/reference/compose-file/services/#cgroup).

If you're using a recent version of Docker Compose, you should also remove the `version` key as it is no longer required.

With that change and a quick `docker compose up -d`, your container metrics should start trickling through to the New Relic UI.
