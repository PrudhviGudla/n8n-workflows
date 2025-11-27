# n8n Workflows

This project is deployed on **Windows** using **Docker Desktop** with a custom bridge network to ensure secure communication between containers.

### 1. Data Persistence & Database Strategy
Unlike standard ephemeral Docker setups, this architecture prioritizes data safety and separation of concerns:
* **System Database (n8n):** Uses the default **SQLite** database. Crucially, I implemented a **Docker Volume** (`n8n_data`) to persist the `/home/node/.n8n` directory. This ensures that even if the container is destroyed (to update env vars or version), workflows, credentials, and user logins are preserved.
* **Database (PostgreSQL):** A separate container running on the same bridge network, acting as the target for the AI agent to query.

### 2. Connectivity & Webhooks (ngrok Service)
To enable external triggers (like WhatsApp webhooks) on a local machine, I configured **ngrok** as a background Windows Service.

**The Challenge:**
Running ngrok manually means the URL changes every time the terminal closes.
**The Solution:**
I configured ngrok to run as a **Windows Service** pointing to a static domain, ensuring 24/7 uptime and a permanent URL.

#### Setup ngrok:
1.  **Static Domain:** Claimed a free static domain from the ngrok dashboard (e.g., `my-agent.ngrok-free.app`).
2.  **Configuration:** Created `C:\ngrok\ngrok.yml` with the following configuration:

    ```yaml
    version: "3"
    agent:
      authtoken: 
    endpoints:
      - name: n8n
        url: 
        upstream:
          url: 5678
    ```
3.  **Service Installation:**

    Make sure you installed the ngrok executable via the zip file, and it is added to the path (in case of Windows)

    ```
    # Install and Start as a background service
    ngrok service install --config "C:\ngrok\ngrok.yml"
    ngrok service start
    ```

## Installation & Setup
Use these commands to spin up the network and containers. Note that n8n is linked to the n8n_data volume to preserve your login.

### 1. Create Network

```
docker network create n8n-network
```

### 2. Run Postgres (If needed a Target DB for connecting to workflows)

```
docker run -d --name postgres --network n8n-network --restart unless-stopped \
  -e POSTGRES_USER=n8n -e POSTGRES_PASSWORD=n8n_password -e POSTGRES_DB=n8n \
  -v postgres_data:/var/lib/postgresql/data postgres:16-alpine
```

### 3. Run n8n (Linked to ngrok & Postgres)

```
# Replace WEBHOOK_URL with your actual ngrok domain
docker run -d --name n8n --network n8n-network -p 5678:5678 \
  -e WEBHOOK_URL="https://your-domain.ngrok-free.app" \
  -v n8n_data:/home/node/.n8n \
  --restart unless-stopped n8nio/n8n
```

## Repository Structure
- /workflows: Contains the Main Agent and Tool sub-workflows.
