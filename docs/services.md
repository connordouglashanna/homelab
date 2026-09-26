# Services

This file exists to track plans for expansion into additional services being hosted on the homelab. Current or primary services will be added to the main README in the root directory. 

# TODO services

## AI 

### Ollama

As the device network grows, there's a potential for extra RAM to be allocated to running extremely small 'personal assistant' utility models on the device for use by n8n and similar services. [Ollama](https://ollama.com/) is a convenient tool for achieving this at this scale. If I'd ever like to swap to [larger models](https://huggingface.co/spaces/open-llm-leaderboard/open_llm_leaderboard#/), I'd be better off running the models on my primary desktop and orchestrating access through the homelab network. 

### n8n

[n8n](https://n8n.io/)
is a leading workflow automation tool, and dovetails nicely with the availability of small LLMs mentioned above.

## Networking

### Meshtastic

I heard about some guy running Meshtastic at [Burning Man](https://www.burningmesh.org/), and that sounded pretty cool. It'd be fun to tinker with setting this up for guest access at home, or as a portable festival setup for camping, road trips, and so on.

### Traefik/nginx/Caddy

A reverse proxy will be needed in order to safely expose the homelab to the internet, even throught a Cloudflare tunnel.

### Cloudflare DDNS

If network exposure over IPv6 is viable, then Cloudflare DDNS would be required in combination with Traefik to resolve the non-static IP address into an updated domain.

### CrowdSec

[Crowdsec](https://doc.crowdsec.net/docs/next/intro) should help. I'm paranoid, so I'll be throwing the kitchen sink at network security.

### Fail2ban

Defeats brute-force port scanners. Shouldn't be an issue unless I use DDNS, but I'll likely implement well beforehand. 

## Monitoring

### Prometheus/Grafana

Did I even build a homelab if I can't use a live monitoring dashboard with alerting to know if the recipe management app is running?

## Household

### Recipe App

Likely RecipeSage or Norish.

### Obsidian Sync

Many options here, but I probably should implement one of them. 

## Other Stuff

See [this repo](https://www.github.com/Haxxnet/Compose-Examples) for details about self-hosting other applications in a homelab context.

