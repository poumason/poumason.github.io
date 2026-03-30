---
title: How to using Openshell to run OpenClaw
date: 2026-03-28 22:39:26
categories: AI
tags:
    - K8s
    - OpenShell
---
Try to understand how about the Openshell architecutre.
<!-- more-->
# How to install
## using local ollama on docker
[Run Local Inference with Ollama](https://docs.nvidia.com/openshell/latest/tutorials/inference-ollama.html)

if you want to use local ollama, must be changed somethings.

- create a provider
```
openshell provider create \
    --name ollama \
    --type openai \
    --credential OPENAI_API_KEY=empty \
    --config OPENAI_BASE_URL=http://host.openshell.internal:11434/v1
```
> OpenShell injects host.openshell.internal so sandboxes and the gateway can reach the host machine. You can also use the host’s LAN IP.

- set inference routing
```
openshell inference set --provider ollama --model qwen3.5:0.8b
```
if you want to using other model, must be changed.


because the ollama be run in host docker.

## OpenClaw

- how to solve port foward
```
openshell forward start 18789 my-openclaw
```

- openclaw.json
```
{
  "wizard": {
    "lastRunAt": "2026-03-29T14:40:04.962Z",
    "lastRunVersion": "2026.3.11",
    "lastRunCommand": "onboard",
    "lastRunMode": "local"
  },
  "models": {
    "mode": "merge",
    "providers": {
      "ollama": {
        "baseUrl": "https://inference.local",
        "apiKey": "OLLAMA_API_KEY",
        "api": "ollama",
        "models": [
          {
            "id": "glm-4.7-flash",
            "name": "glm-4.7-flash",
            "reasoning": false,
            "input": [
              "text"
            ],
            "cost": {
              "input": 0,
              "output": 0,
              "cacheRead": 0,
              "cacheWrite": 0
            },
            "contextWindow": 128000,
            "maxTokens": 8192
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "ollama/minimax-m2.5:cloud"
      },
      "models": {
        "ollama/glm-4.7-flash": {},
        "ollama/minimax-m2.5:cloud": {}
      },
      "workspace": "/sandbox/.openclaw/workspace"
    }
  },
  "tools": {
    "profile": "coding"
  },
  "commands": {
    "native": "auto",
    "nativeSkills": "auto",
    "restart": true,
    "ownerDisplay": "raw"
  },
  "session": {
    "dmScope": "per-channel-peer"
  },
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "22f72aa5bb549c9d0381a01ffc9bf73d7591861c4d2ece7e"
    },
    "tailscale": {
      "mode": "off",
      "resetOnExit": false
    },
    "nodes": {
      "denyCommands": [
        "camera.snap",
        "camera.clip",
        "screen.record",
        "contacts.add",
        "calendar.add",
        "reminders.add",
        "sms.send"
      ]
    }
  },
  "meta": {
    "lastTouchedVersion": "2026.3.11",
    "lastTouchedAt": "2026-03-29T14:40:04.968Z"
  }
}

```

# References
- [NVIDIA OpenShell Developer Guide](https://docs.nvidia.com/openshell/latest/index.html)
- [NVIDIA/OpenShell](https://github.com/NVIDIA/OpenShell?tab=readme-ov-file)
- [NVIDIA/OpenShell-Community](https://github.com/NVIDIA/OpenShell-Community)
- [Secure Long Running AI Agents with OpenShell on DGX Spark](https://build.nvidia.com/spark/openshell/troubleshooting)