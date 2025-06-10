
### **Project Documentation: AI-Powered Client Intake & Onboarding System**

**Version:** 1.0  
**Date:** Jun 10, 2025
**Author:** Zerubabel  

#### 1. Project Overview

This project addresses the common business challenges of manual client intake. The goal was to design and deploy a fully automated pipeline that captures new leads from a website, uses Artificial Intelligence to qualify and enrich the data, sets up the necessary project infrastructure, and instantly notifies the relevant team members.

The result is a resilient, scalable, and secure system that transforms a slow, error-prone manual process into an instantaneous, intelligent, and reliable workflow. This system saves significant administrative time, ensures no lead is ever missed, and allows the sales team to focus on high-value activities.

#### 2. Key Features & Functionality

*   **Website Integration:** Captures lead data directly from the company's public-facing contact form.
*   **Secure Endpoint:** The data is received via a secure, authenticated webhook, preventing unauthorized access.
*   **AI-Powered Lead Qualification:** Utilizes Google's Gemini AI to analyze the lead's unstructured message to extract key business data:
    *   The specific service requested (e.g., SEO, PPC).
    *   The estimated project size (Small, Medium, Large).
    *   A concise summary of the client's request.
*   **Data Persistence:** All incoming leads and their AI analysis are permanently stored in a professional-grade PostgreSQL database (Supabase).
*   **Intelligent Branching Logic:** The workflow automatically differentiates between low-value and high-value leads.
*   **Dynamic Project Setup:** For high-value leads, the system automatically:
    *   Creates a dedicated client folder in Google Drive.
    *   Creates a new project card in a Trello "Sales Pipeline" board.
    *   Populates the Trello card with all relevant contact info, AI-generated summaries, and a direct link to the Google Drive folder.
*   **Instant Team Notification:** Posts a professionally formatted alert to a dedicated Slack channel (`#new-leads`) for immediate team awareness and action.
*   **Resilient Error Handling:** In case of any API failure or unexpected error, the system triggers a separate error path that instantly notifies an administrator in a `#workflow-errors` Slack channel with specific details about the failure.

#### 3. System Architecture

The system is deployed on the Google Cloud Platform and architected using a modern, containerized approach for stability and security.

**Data Flow Diagram:**

`[User on Website]` -> `HTTPS via n8n.hawaryamedianetworks.com` -> `[Google Cloud VM]`
`...` -> `[Caddy Reverse Proxy (Ports 80/443)]`
`...` -> `[Docker Engine]`
`...` -> `[n8n Container (Private Port 5678)]`
`...` -> **External APIs** `([Gemini], [Supabase], [Google Drive], [Trello], [Slack])`

**Core Technologies Used:**

*   **Cloud Provider:** Google Cloud Platform (Compute Engine)
*   **Containerization:** Docker & Docker Compose
*   **Reverse Proxy & SSL:** Caddy Server (for automatic HTTPS)
*   **Automation Engine:** n8n (self-hosted)
*   **Database:** Supabase (PostgreSQL)
*   **Artificial Intelligence:** Google Gemini
*   **Project Management:** Trello
*   **Collaboration:** Slack, Google Drive

#### 4. Infrastructure & Deployment Details

The entire application stack is defined and managed via Docker Compose for reproducibility and ease of management.

**File: `docker-compose.yml`**
```yml
version: '3.7'

services:
  n8n:
    image: n8nio/n8n
    restart: always
    environment:
      # Sets the correct timezone for scheduling
      - GENERIC_TIMEZONE=Africa/Addis_Ababa
      # Limits RAM usage for stability on a free-tier VM
      - NODE_OPTIONS=--max-old-space-size=512
      # IMPORTANT: Sets n8n's public URL for webhooks and OAuth callbacks
      - WEBHOOK_URL=https://n8n.hawaryamedianetworks.com/
    volumes:
      # Persists all n8n workflow and credential data
      - n8n_data:/home/node/.n8n

  caddy:
    image: caddy:latest
    restart: always
    ports:
      # Exposes the public HTTP and HTTPS ports
      - "80:80"
      - "443:443"
      - "443:443/udp" 
    volumes:
      # Links the Caddyfile configuration
      - ./Caddyfile:/etc/caddy/Caddyfile
      # Persists Caddy's data and SSL certificates
      - caddy_data:/data
      - caddy_config:/config

volumes:
  n8n_data:
  caddy_data:
  caddy_config:
```

**File: `Caddyfile`**
This file configures the Caddy reverse proxy.
```
# Defines the public domain Caddy will manage
n8n.hawaryamedianetworks.com {
    # Forwards all traffic internally to the n8n service on its private port
    reverse_proxy n8n:5678
}
```

**Firewall Configuration:**
The Google Cloud VM instance is secured with a firewall rule that only allows public inbound traffic on the standard web ports, applied via a network tag (`n8n-server-tag`).
*   **Allow TCP:** Port `80` (for HTTP and SSL certificate validation)
*   **Allow TCP:** Port `443` (for all HTTPS traffic)
*   All other ports, including the n8n application port `5678`, are **not** exposed to the public internet.

#### 5. Management & Maintenance

The system is managed via SSH connection to the Google Cloud VM. All commands are run from the project directory (`~/n8n-caddy`).

*   **Check Status:** `docker-compose ps`
*   **Stop Services:** `docker-compose down`
*   **Start Services:** `docker-compose up -d`
*   **View Logs (n8n):** `docker-compose logs n8n`
*   **View Logs (Caddy):** `docker-compose logs caddy`

**Updating n8n:**
To update to a newer version of n8n, follow this process:
```bash
# 1. Pull the latest n8n image from Docker Hub
docker-compose pull n8n

# 2. Stop and remove the old containers
docker-compose down

# 3. Relaunch the stack. Docker Compose will use the new image automatically.
docker-compose up -d
```
