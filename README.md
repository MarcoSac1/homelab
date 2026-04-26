🏠 Homelab Infrastructure
Infrastruttura modulare e sicura per un piccolo gruppo di utenti (3–4 persone), costruita su Ubuntu con Docker Compose. Tutti i servizi interni sono accessibili esclusivamente tramite autenticazione centralizzata con Authentik (SSO via OpenID Connect / OAuth2), senza esporre nulla direttamente su Internet.

L'accesso remoto avviene tramite WireGuard VPN. Ogni utente, una volta autenticato, può accedere solo ai servizi e alle risorse previste dal proprio ruolo.

🗂️ Struttura del Repository
text
homelab/
├── README.md
├── dns/
│   └── pihole/
│       ├── docker-compose.yml
│       └── .env.example
└── stacks/
    ├── authentik/
    │   ├── docker-compose.yml
    │   └── .env.example
    ├── wireguard/
    │   ├── docker-compose.yml
    │   └── .env.example
    ├── ai/
    │   ├── ollama/docker-compose.yml
    │   └── open-webui/
    │       ├── docker-compose.yml
    │       └── .env.example
    ├── portainer/docker-compose.yml
    ├── uptime-kuma/docker-compose.yml
    └── reverse-proxy/
        ├── docker-compose.yml
        └── Caddyfile
🧩 Servizi
Servizio	Immagine	Scopo
Caddy	caddy:2-alpine	Reverse proxy HTTPS
Authentik	ghcr.io/goauthentik/server:2025.8.4	SSO / IdP centralizzato
WireGuard (wg-easy)	ghcr.io/wg-easy/wg-easy	VPN accesso remoto
Open WebUI	ghcr.io/open-webui/open-webui:main	Frontend AI locale
Ollama	ollama/ollama:latest	Backend LLM locale
Portainer CE	portainer/portainer-ce:latest	Gestione container
Uptime Kuma	louislam/uptime-kuma:1	Monitoring e status page
Pi-hole	pihole/pihole:latest	DNS locale + ad-blocking
PostgreSQL 16	postgres:16-alpine	Database Authentik
Redis	redis:alpine	Cache Authentik
👤 Utenti e Accessi
La gestione utenti è centralizzata in Authentik tramite local user store (nessun LDAP/AD). I ruoli sono implementati con gruppi e policy binding sulle singole applicazioni.

Ruolo	Portainer	Open WebUI	Uptime Kuma	Pi-hole	wg-easy
Ruolo	Portainer	Open WebUI	Uptime Kuma	Pi-hole	wg-easy
admin	Accesso completo	✅	Dashboard completa	✅	✅
studente	Crea/gestisce i propri container	✅	Solo status page	❌	❌
guest	❌	❌	Solo status page	❌	❌
Dettaglio ruolo studente in Portainer
Gli studenti accedono con ruolo User (non Admin) e possono: creare e gestire i propri container e stack, visualizzare log e metriche dei propri container. Non possono accedere agli stack di sistema, interagire con container di altri utenti o modificare reti Docker globali.

Status page
Studenti e guest accedono alla status page di Uptime Kuma tramite un'applicazione dedicata in Authentik, senza mai vedere il pannello admin.

🔒 Sicurezza
SSH: solo chiavi pubbliche (PasswordAuthentication no)

WireGuard: unico ingresso remoto, nessun servizio esposto su Internet

Caddy: HTTPS automatico su tutti i servizi interni

Authentik: nessun servizio raggiungibile senza autenticazione SSO

Segreti: nessuna password nel repository, solo .env.example

Firewall UFW: 22/tcp, 80/tcp, 443/tcp, 1194/udp, 51821/tcp

🚀 Implementazioni Future
Ambienti di ricerca isolati: ogni studente potrà deployare container dedicati per ricerche OSINT, analisi dati, scraping o test di sicurezza in sandbox, gestiti autonomamente tramite Portainer

Isolamento rete: ogni ambiente di ricerca su rete Docker dedicata, separata dai servizi di sistema

Autenticazione granulare: estensione delle policy Authentik per accesso a specifici ambienti per singolo utente

Logging centralizzato: stack Loki/Grafana o ELK per raccogliere log di tutti i container

Backup automatico: schedulazione periodica di backup del database Authentik e dei volumi critici
