# Installazione Docker Registry v3 su QNAP Container Station

**Versione:** 1.2  
**Data:** 29 Novembre 2025  
**Container Station:** 3.1.1.1451+  
**Registry:** 3.0.0

---

## Panoramica

Questa guida descrive come installare e configurare un Docker Registry privato su QNAP NAS utilizzando Container Station. La procedura utilizza la CLI Docker per garantire un corretto port mapping, aggirando le limitazioni note dell'interfaccia grafica di Container Station.

---

## Prerequisiti

- QNAP NAS con Container Station 3.x installato
- Accesso SSH al NAS abilitato
- Client SSH (Terminal su macOS/Linux, PuTTY su Windows)

---

## Architettura Storage del Registry

### Struttura Directory

Il registry organizza i dati in modo content-addressable:

```
/share/Container/registry-data/
└── docker/
    └── registry/
        └── v2/
            ├── blobs/
            │   └── sha256/
            │       ├── 00/
            │       │   └── 00a123.../
            │       │       └── data        ← Layer binario
            │       ├── 01/
            │       └── ff/
            └── repositories/
                └── <nome-immagine>/
                    ├── _layers/
                    │   └── sha256/
                    │       └── <digest>/
                    │           └── link     ← Puntatore a blob
                    ├── _manifests/
                    │   ├── revisions/
                    │   └── tags/
                    │       └── <tag>/
                    │           └── current/
                    │               └── link ← Tag corrente
                    └── _uploads/            ← Upload temporanei
```

### Componenti

| Directory | Contenuto | Caratteristica |
|-----------|-----------|----------------|
| `blobs/sha256/` | Layer binari | Content-addressable, deduplicati |
| `repositories/` | Metadati | Link ai blob |
| `_manifests/tags/` | Mapping tag → digest | Risoluzione tag |
| `_uploads/` | Upload in corso | Temporaneo |

### Vantaggi per Snapshot QNAP

1. **Deduplicazione naturale**: Layer condivisi tra immagini esistono una sola volta
2. **Snapshot incrementali efficienti**: Solo i nuovi blob vengono catturati
3. **Consistenza garantita**: I blob sono immutabili (write-once)

### Separazione Container / Dati

| Dentro il container (effimero) | Fuori dal container (persistente) |
|--------------------------------|-----------------------------------|
| Binario `/bin/registry` | Blob immagini |
| Config default | Repository metadata |
| Cache in-memory | Tag e manifest |

**Aggiornamento container**: I dati nel volume montato rimangono intatti. Il nuovo container legge gli stessi dati.

---

## Procedura di Installazione

### 1. Connessione SSH al NAS

```bash
ssh admin@<IP_NAS>
```

### 2. Creazione Directory per i Dati

```bash
mkdir -p /share/Container/registry-data
```

### 3. Rimozione Container Esistente (se presente)

```bash
docker stop registry
docker rm registry
```

### 4. Creazione Container Registry v3

#### Configurazione Base

Configurazione minima per test e sviluppo:

```bash
docker run -d \
  --name registry \
  --restart=always \
  -p 5000:5000 \
  -e OTEL_TRACES_EXPORTER=none \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:5000 \
  -v /share/Container/registry-data:/var/lib/registry \
  registry:3
```

#### Configurazione Intermedia (Consigliata)

Con chiave segreta persistente, limiti risorse e gestione log:

```bash
# 1. Genera chiave segreta persistente
openssl rand -hex 32 > /share/Container/registry-secret.txt

# 2. Avvia container
docker run -d \
  --name registry \
  --restart=always \
  -p 5000:5000 \
  --memory=1g \
  --memory-swap=1g \
  --cpus=2 \
  --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  -e OTEL_TRACES_EXPORTER=none \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:5000 \
  -e REGISTRY_HTTP_SECRET=$(cat /share/Container/registry-secret.txt) \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  -v /share/Container/registry-data:/var/lib/registry \
  registry:3
```

#### Configurazione Produzione (con Redis)

Per ambienti con alto traffico e molti pull concorrenti:

```bash
# 1. Genera chiave segreta persistente
openssl rand -hex 32 > /share/Container/registry-secret.txt

# 2. Avvia Redis per cache
docker run -d \
  --name registry-redis \
  --restart=always \
  --memory=256m \
  redis:alpine

# 3. Avvia Registry con cache Redis
docker run -d \
  --name registry \
  --restart=always \
  --link registry-redis:redis \
  -p 5000:5000 \
  --memory=1g \
  --memory-swap=1g \
  --cpus=2 \
  --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  -e OTEL_TRACES_EXPORTER=none \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:5000 \
  -e REGISTRY_HTTP_SECRET=$(cat /share/Container/registry-secret.txt) \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  -e REGISTRY_STORAGE_CACHE_BLOBDESCRIPTOR=redis \
  -e REGISTRY_REDIS_ADDR=redis:6379 \
  -v /share/Container/registry-data:/var/lib/registry \
  registry:3
```

### Confronto Configurazioni

| Parametro | Base | Intermedia | Produzione |
|-----------|------|------------|------------|
| Limiti risorse | No | Sì | Sì |
| Chiave persistente | No | Sì | Sì |
| Gestione log | No | Sì | Sì |
| Cancellazione immagini | No | Sì | Sì |
| Cache Redis | No | No | Sì |
| Uso consigliato | Test | Uso normale | Alto traffico |

### 5. Verifica Installazione

```bash
# Container in esecuzione
docker ps | grep registry

# Port mapping
docker port registry
# Output: 5000/tcp -> 0.0.0.0:5000

# Log puliti
docker logs registry
# Output: level=info msg="listening on 0.0.0.0:5000"
```

### 6. Test Funzionamento

```bash
# Test locale
curl http://localhost:5000/v2/

# Test remoto
curl http://<IP_NAS>:5000/v2/

# Risposta attesa: {}
```

---

## Configurazione Client Docker

### Linux

Modificare `/etc/docker/daemon.json`:

```json
{
  "insecure-registries": ["<IP_NAS>:5000"]
}
```

Riavviare Docker:

```bash
sudo systemctl restart docker
```

### macOS / Windows (Docker Desktop)

Docker Desktop → Settings → Docker Engine → aggiungere:

```json
{
  "insecure-registries": ["<IP_NAS>:5000"]
}
```

Apply & Restart.

---

## Utilizzo del Registry

### Push immagine

```bash
docker tag alpine:latest <IP_NAS>:5000/alpine:latest
docker push <IP_NAS>:5000/alpine:latest
```

### Pull immagine

```bash
docker pull <IP_NAS>:5000/alpine:latest
```

### Elenco immagini

```bash
curl http://<IP_NAS>:5000/v2/_catalog
```

### Elenco tag

```bash
curl http://<IP_NAS>:5000/v2/<nome-immagine>/tags/list
```

---

## Backup con Snapshot QNAP

### Strategia Consigliata

1. **Snapshot schedulata** su `/share/Container/` ogni 4-6 ore
2. **Replica snapshot** verso secondo NAS o cloud
3. **Retention**: mantieni N snapshot per point-in-time recovery

### Consistenza Snapshot

I blob sono immutabili, quindi le snapshot sono sempre consistenti. Per snapshot critiche su registry ad alto traffico:

```bash
# Pausa temporanea (opzionale)
docker pause registry
# → Esegui snapshot QNAP
docker unpause registry
```

### Cosa Includere nel Backup

| Path | Contenuto | Priorità |
|------|-----------|----------|
| `/share/Container/registry-data/` | Immagini e metadati | **Critico** |
| `/share/Container/registry-secret.txt` | Chiave segreta | **Critico** |

### Restore

```bash
# 1. Ripristina snapshot QNAP
# 2. Ricrea container con stesso comando (sezione 4)
# 3. Verifica: curl http://localhost:5000/v2/_catalog
```

---

## Ottimizzazioni QNAP

### Storage

- Usa **SSD/NVMe** per `/share/Container/`
- Abilita **SSD Cache** in Storage & Snapshots
- Sposta **Docker Root su SSD** in Container Station → Preferences

### Rete

- **Jumbo Frame 9000** se supportato dalla rete
- **Host Network Mode** (`--network=host`) per eliminare overhead NAT

### Sistema

- Disabilita **servizi QTS non necessari**
- Escludi `/share/Container/` dall'**antivirus**
- Espandi **RAM a 8GB+** per uso intensivo

### Verifica Storage Driver

```bash
docker info | grep "Storage Driver"
# Ottimale: overlay2
```

---

## Manutenzione

### Garbage Collection

Recupera spazio eliminando layer orfani:

```bash
# Dry-run (verifica)
docker exec registry bin/registry garbage-collect \
  --dry-run /etc/distribution/config.yml

# Esecuzione
docker exec registry bin/registry garbage-collect \
  /etc/distribution/config.yml
```

### Cancellazione Immagine

```bash
# Ottieni digest
DIGEST=$(curl -s -I -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  http://<IP_NAS>:5000/v2/<immagine>/manifests/<tag> | \
  grep -i docker-content-digest | awk '{print $2}' | tr -d '\r')

# Cancella
curl -X DELETE http://<IP_NAS>:5000/v2/<immagine>/manifests/$DIGEST

# Garbage collection
docker exec registry bin/registry garbage-collect /etc/distribution/config.yml
```

### Aggiornamento Registry

```bash
docker pull registry:3
docker stop registry
docker rm registry
# Esegui nuovamente il comando docker run della configurazione scelta
```

---

## Troubleshooting

### Porta non raggiungibile

1. Verifica firewall: Control Panel → Security
2. Aggiungi `10.0.0.0/8` alle reti consentite

### Errore HTTPS

Configura `insecure-registries` sul client (sezione Configurazione Client).

### Verifica rete

```bash
netstat -tlnp | grep 5000
# Output: tcp 0 0 0.0.0.0:5000 ... docker-proxy
```

### Monitoraggio risorse

```bash
docker stats registry
```

---

## Riferimenti

- [CNCF Distribution Documentation](https://distribution.github.io/distribution/)
- [Docker Registry API v2](https://docs.docker.com/registry/spec/api/)
- [QNAP Container Station](https://www.qnap.com/en/software/container-station)
