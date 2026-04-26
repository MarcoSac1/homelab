# 🏠 Homelab Infrastructure

Infrastruttura modulare e sicura per un piccolo gruppo di utenti (3–4 persone), costruita su Ubuntu con Docker Compose. Tutti i servizi interni sono accessibili esclusivamente tramite autenticazione centralizzata con **Authentik** (SSO via OpenID Connect / OAuth2), senza esporre nulla direttamente su Internet.

L'accesso remoto avviene tramite **WireGuard VPN**. Ogni utente, una volta autenticato, può accedere solo ai servizi e alle risorse previste dal proprio ruolo.

---

## 🖥️ Infrastruttura Hardware e Configurazione Iniziale

Il progetto è stato sviluppato su **Ubuntu Server** installato in una macchina virtuale con le seguenti caratteristiche:

| Componente | Dettaglio |
|---|---|
| **CPU** | Intel Core i7 — 8ª generazione, 6 vCPU |
| **RAM** | 12 GB |
| **OS** | Ubuntu Server 24.04 LTS |
| **Tipo** | Macchina virtuale |
| **IP statico** | `192.168.1.50` |

### Configurazione di rete

Inizialmente la scheda di rete era configurata in modalità **NAT**, ma questa impostazione non permetteva la corretta esposizione del servizio VPN verso l'esterno. La configurazione è stata modificata assegnando alla VM una scheda di rete in modalità **bridge**, con un indirizzo IP statico (`192.168.1.50`) per garantire raggiungibilità e stabilità all'interno della rete locale.

Dopo la predisposizione della rete, è stato installato **Docker** per containerizzare i servizi del laboratorio, seguendo un approccio modulare e facilmente riproducibile. Successivamente è stato installato e configurato **WireGuard**, con i profili client necessari per l'accesso remoto sicuro dall'esterno della rete locale.

---

## 🌐 DNS Locale e Scelta del Dominio

Per i domini interni del laboratorio è stato adottato il suffisso **`.home.arpa`**, in conformità con il **RFC 8375** ("Special-Use Domain Name \'home.arpa\'"), che riserva questo dominio esclusivamente per reti residenziali e di laboratorio locali, senza rischio di conflitti con domini pubblici su Internet.

Questa scelta garantisce:
- **Nessun conflitto** con domini pubblici registrati
- **Conformità agli standard IETF** per reti private
- **Compatibilità** con resolver DNS moderni che riconoscono `.home.arpa` come dominio non instradabile su Internet

La risoluzione dei nomi interni è gestita da **Pi-hole**, che funge da DNS autoritativo locale per tutti i servizi del laboratorio.

---

## 🗂️ Struttura del Repository

```
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
```

---

## 🧩 Servizi

| Servizio | Immagine | Scopo |
|---|---|---|
| Caddy | `caddy:2-alpine` | Reverse proxy HTTPS |
| Authentik | `ghcr.io/goauthentik/server:2025.8.4` | SSO / IdP centralizzato |
| WireGuard (wg-easy) | `ghcr.io/wg-easy/wg-easy` | VPN accesso remoto |
| Open WebUI | `ghcr.io/open-webui/open-webui:main` | Frontend AI locale |
| Ollama | `ollama/ollama:latest` | Backend LLM locale |
| Portainer CE | `portainer/portainer-ce:latest` | Gestione container |
| Uptime Kuma | `louislam/uptime-kuma:1` | Monitoring e status page |
| Pi-hole | `pihole/pihole:latest` | DNS locale + ad-blocking |
| PostgreSQL 16 | `postgres:16-alpine` | Database Authentik |
| Redis | `redis:alpine` | Cache Authentik |

---

## 👤 Utenti e Accessi

La gestione utenti è centralizzata in Authentik tramite **local user store** (nessun LDAP/AD). I ruoli sono implementati con gruppi e policy binding sulle singole applicazioni.

| Ruolo | Portainer | Open WebUI | Uptime Kuma | Pi-hole | wg-easy |
|---|---|---|---|---|---|
| `admin` | Accesso completo | ✅ | Dashboard completa | ✅ | ✅ |
| `studente` | Crea/gestisce i propri container | ✅ | Solo status page | ❌ | ❌ |
| `guest` | ❌ | ❌ | Solo status page | ❌ | ❌ |

### Dettaglio ruolo studente in Portainer

Gli studenti accedono con ruolo **User** (non Admin) e possono:
- Creare e gestire i propri container e stack
- Visualizzare log e metriche dei propri container

Non possono:
- Accedere o modificare gli stack di sistema
- Interagire con container di altri utenti
- Modificare reti Docker globali

### Status page

Studenti e guest accedono alla status page di Uptime Kuma tramite un\'applicazione dedicata in Authentik, senza mai vedere il pannello admin.

---

## 🔑 Setup Iniziale

```bash
# 1. Aggiornamento sistema
sudo apt update && sudo apt upgrade -y

# 2. Impostazione IP statico (adatta l\'interfaccia di rete)
sudo nano /etc/netplan/00-installer-config.yaml

# 3. Installazione Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER

# 4. Clona il repository
git clone https://github.com/MarcoSac1/homelab.git
cd homelab

# 5. Crea i file .env dai template
cp stacks/authentik/.env.example stacks/authentik/.env
cp stacks/wireguard/.env.example stacks/wireguard/.env
cp stacks/ai/open-webui/.env.example stacks/ai/open-webui/.env
cp dns/pihole/.env.example dns/pihole/.env

# 6. Genera il secret key per Authentik
openssl rand -base64 50

# 7. Avvia i servizi
cd stacks/authentik   && docker compose up -d
cd ../reverse-proxy   && docker compose up -d
cd ../wireguard       && docker compose up -d
cd ../uptime-kuma     && docker compose up -d
cd ../portainer       && docker compose up -d
cd ../ai/ollama       && docker compose up -d
cd ../ai/open-webui   && docker compose up -d
cd ../../dns/pihole   && docker compose up -d
```

---

## 🔒 Sicurezza

- **SSH**: solo chiavi pubbliche (`PasswordAuthentication no`)
- **WireGuard**: unico ingresso remoto, nessun servizio esposto direttamente su Internet
- **Caddy**: HTTPS automatico su tutti i servizi interni
- **Authentik**: nessun servizio raggiungibile senza autenticazione SSO
- **Segreti**: nessuna password nel repository, solo `.env.example`
- **Firewall UFW**: `22/tcp`, `80/tcp`, `443/tcp`, `1194/udp`, `51821/tcp`

---

## 🚀 Implementazioni Future

- **Ambienti di ricerca isolati**: ogni studente potrà deployare container dedicati per ricerche OSINT, analisi dati, scraping o test di sicurezza in sandbox, gestiti autonomamente tramite Portainer
- **Isolamento rete**: ogni ambiente di ricerca su rete Docker dedicata, separata dai servizi di sistema
- **Autenticazione granulare**: estensione delle policy Authentik per accesso a specifici ambienti per singolo utente
- **Logging centralizzato**: stack Loki/Grafana o ELK per raccogliere log di tutti i container
- **Backup automatico**: schedulazione periodica di backup del database Authentik e dei volumi critici
'''
