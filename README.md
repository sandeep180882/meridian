# Event POC — Docker Edition
### No AKS · No load balancer · One VM · docker compose up

---

## What runs where

```
Azure Linux VM (Standard_B2s — 2 vCPU, 4 GB)
└── Docker Engine
    ├── nginx              ← reverse proxy, port 80/443 (only public port)
    ├── orchestrator       ← FastAPI, UI, routing, GPT-4
    ├── consumer           ← Event Hub listener, nudges
    ├── billing-agent      ← invoices + payment failures
    ├── account-agent      ← activations + account status
    ├── notification-agent ← nudge history
    └── postgres           ← persistent storage (volume-mounted)

All containers talk via Docker bridge network (eventpoc-net).
Service names are the DNS hostnames — orchestrator reaches
billing-agent at http://billing-agent:8001 automatically.
```

---

## Project Layout

```
├── docker-compose.yml          Single file that defines everything
├── .env.example                Copy to .env and fill in
├── nginx/
│   ├── nginx.conf
│   └── conf.d/eventpoc.conf   Reverse proxy config
├── services/
│   ├── orchestrator/
│   │   ├── Dockerfile
│   │   ├── main.py            FastAPI app + webhook handlers
│   │   ├── llm.py             Azure OpenAI calls
│   │   ├── event_hub.py       Event Hub producer
│   │   └── validator.py
│   ├── consumer/
│   │   ├── Dockerfile
│   │   ├── main.py            Event Hub consumer loop
│   │   └── nudge.py           Twilio WhatsApp/SMS/Voice
│   └── agents/
│       ├── Dockerfile          One Dockerfile for ALL agents (build arg)
│       ├── base_agent.py       BaseAgent class — extend to add new agents
│       ├── billing/main.py
│       ├── account/main.py
│       └── notification/main.py
├── shared/
│   ├── schemas.py             Pydantic models
│   ├── db.py                  SQLAlchemy + PostgreSQL
│   └── config.py              Settings
├── static/index.html          Web UI
├── data/*.csv                 Sample event data
└── scripts/
    ├── vm-setup.sh            Run once on fresh VM
    ├── deploy.sh              Build + start everything
    ├── add-agent.sh           Scaffold a new agent
    └── ops.sh                 Common ops commands
```

---

## Step-by-step deployment

### 1. Create the VM (Azure Portal or CLI)

```bash
az group create --name eventpoc-rg --location eastus

az vm create \
  --resource-group eventpoc-rg \
  --name eventpoc-vm \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard

# Open ports 80 and 443
az vm open-port --resource-group eventpoc-rg --name eventpoc-vm --port 80 --priority 100
az vm open-port --resource-group eventpoc-rg --name eventpoc-vm --port 443 --priority 110
```

### 2. SSH into the VM and install Docker

```bash
# Get the VM public IP
az vm show -d -g eventpoc-rg -n eventpoc-vm --query publicIps -o tsv

ssh azureuser@<vm-public-ip>

# On the VM:
curl -fsSL https://raw.githubusercontent.com/<you>/<repo>/main/scripts/vm-setup.sh | bash

# Log out and back in for docker group
exit
ssh azureuser@<vm-public-ip>
```

### 3. Copy the project to the VM

```bash
# From your local machine:
scp -r ./event-poc-docker azureuser@<vm-public-ip>:~/event-poc
ssh azureuser@<vm-public-ip>
cd ~/event-poc
```

### 4. Configure and deploy

```bash
cp .env.example .env
nano .env          # fill in Event Hub, OpenAI, Twilio credentials

chmod +x scripts/*.sh
./scripts/deploy.sh
```

### 5. Verify

```bash
docker compose ps          # all should show "healthy" or "running"
curl http://localhost/health
# Open http://<vm-public-ip> in browser
```

---

## Adding a new agent

```bash
./scripts/add-agent.sh fraud 8004
# Follow the 3 printed steps (add URL to config + registry)

docker compose build fraud-agent
docker compose up -d fraud-agent
# Existing containers keep running — zero downtime
```

---

## Cost (Docker VM approach)

| Service | Config | Monthly |
|---|---|---|
| Azure VM Standard_B2s | 2 vCPU, 4 GB | ~$35 |
| Azure Event Hub Basic | 1 TU | ~$10 |
| Azure OpenAI GPT-4 | S0, light usage | ~$15 |
| Twilio | Trial credit / PAYG | Free → ~$10 |
| **Total** | | **~$60 / month** |

PostgreSQL, NGINX, and all agents run inside the VM — no extra cost.
No AKS ($0 saved), no Load Balancer ($18 saved), no ACR ($5 saved).

Compared to AKS approach: **~$57/month cheaper**.

---

## Useful commands

```bash
docker compose logs -f                        # live logs all containers
docker compose logs -f orchestrator           # one container
docker compose restart billing-agent          # restart one without touching others
docker compose up -d --no-deps orchestrator   # redeploy one after code change
docker compose exec postgres psql -U eventpoc eventpoc  # query DB directly
docker stats --no-stream                      # CPU/memory per container
```
