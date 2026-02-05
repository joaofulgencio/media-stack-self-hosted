# Media Stack - Setup

## Pre-requisitos

- Docker e Docker Compose instalados
- **Linux**: Docker Engine
- **Windows**: Docker Desktop com backend WSL2

## Passo a passo

### 1. Criar a rede Docker

```bash
docker network create mynetwork
```

### 2. Configurar o .env

```bash
cp .env.example .env
```

Edite o `.env` e preencha os caminhos das suas pastas de midia.

**Linux:**
```
DOWNLOADS_PATH=/mnt/nvme/Downloads
MOVIES_PATH=/mnt/nvme/Filmes
TV_PATH=/mnt/nvme/Series
MEDIA_PATH=/mnt/nvme
```

**Windows:**
```
DOWNLOADS_PATH=D:\Downloads
MOVIES_PATH=D:\Filmes
TV_PATH=D:\Series
MEDIA_PATH=D:\
```

### 3. Configurar GPU (transcoding no Jellyfin)

O compose vem configurado para NVIDIA. Ajuste conforme sua GPU:

#### NVIDIA (padrao do compose)

Nao precisa alterar o compose. Pre-requisitos:

- **Linux**: instalar [nvidia-container-toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
- **Windows**: driver NVIDIA atualizado (WSL2 ja tem suporte nativo)

#### Intel (Quick Sync)

Remova o bloco `deploy:` do jellyfin e substitua por `devices`:

```yaml
  jellyfin:
    # ... (manter o resto)
    devices:
      - /dev/dri:/dev/dri
    # Remova as variaveis NVIDIA_VISIBLE_DEVICES e NVIDIA_DRIVER_CAPABILITIES
    # Remova todo o bloco deploy: resources: reservations:
```

No Jellyfin, ative o transcoding em: Dashboard > Playback > Transcoding > Hardware acceleration: **Intel QuickSync (QSV)** ou **Video Acceleration API (VA-API)**.

Pre-requisitos:
- **Linux**: o usuario precisa ter permissao no device (`sudo usermod -aG render $USER`)
- **Windows**: Quick Sync funciona automaticamente via Docker Desktop + WSL2 com driver Intel atualizado

#### AMD (VA-API)

Mesma configuracao do Intel, usando `devices`:

```yaml
  jellyfin:
    # ... (manter o resto)
    devices:
      - /dev/dri:/dev/dri
    # Remova as variaveis NVIDIA_VISIBLE_DEVICES e NVIDIA_DRIVER_CAPABILITIES
    # Remova todo o bloco deploy: resources: reservations:
```

No Jellyfin, ative: Dashboard > Playback > Transcoding > Hardware acceleration: **Video Acceleration API (VA-API)**.

Pre-requisitos:
- **Linux**: instalar drivers Mesa (`sudo apt install mesa-va-drivers`) e permissao no device (`sudo usermod -aG render,video $USER`)
- **Windows**: suporte limitado. AMD no Docker Desktop/WSL2 nao tem passthrough de GPU confiavel. Recomenda-se transcoding via CPU ou usar Linux.

#### Sem GPU dedicada

Remova do servico `jellyfin`:
- As variaveis `NVIDIA_VISIBLE_DEVICES` e `NVIDIA_DRIVER_CAPABILITIES`
- Todo o bloco `deploy:`

O Jellyfin faz transcoding via CPU (mais lento, mas funciona).

### 4. Subir os servicos base

```bash
docker compose --profile no-vpn up -d qbittorrent radarr sonarr prowlarr jellyfin jellyseerr bazarr
```

### 5. Configurar os servicos

Acesse cada servico no navegador e faca o setup inicial:

| Servico     | URL                      | O que configurar                          |
|-------------|--------------------------|-------------------------------------------|
| Jellyfin    | http://SEU_IP:8096       | Criar conta, adicionar bibliotecas        |
| Radarr      | http://SEU_IP:7878       | Root folder: `/movies`, download client: qBittorrent (host: `qbittorrent`, porta: `5080`) |
| Sonarr      | http://SEU_IP:8989       | Root folder: `/tv`, download client: qBittorrent (host: `qbittorrent`, porta: `5080`) |
| Prowlarr    | http://SEU_IP:9696       | Adicionar indexers, conectar com Radarr/Sonarr |
| qBittorrent | http://SEU_IP:5080       | Trocar senha padrao (admin/adminadmin) |
| Jellyseerr  | http://SEU_IP:5055       | Conectar com Jellyfin, adicionar Radarr/Sonarr e marcar como **Default Server** |
| Bazarr      | http://SEU_IP:6767       | Conectar com Radarr (host: `radarr`) e Sonarr (host: `sonarr`), configurar providers de legenda |

**Importante no Jellyseerr:** ao adicionar Radarr/Sonarr, use o hostname do container (ex: `radarr`, `sonarr`) e marque como **Default Server**. NAO marque como "4K Server" se tiver apenas uma instancia de cada.

### 6. Pegar as API keys e atualizar o .env

Apos configurar Radarr, Sonarr e Jellyseerr, pegue as API keys:

- **Radarr**: Settings > General > API Key
- **Sonarr**: Settings > General > API Key
- **Jellyseerr**: Settings > General > API Key

Cole no `.env`:

```
RADARR_API_KEY=sua_key
SONARR_API_KEY=sua_key
JELLYSEERR_API_KEY=sua_key
```

### 7. Subir Recyclarr e ListSync

```bash
docker compose --profile no-vpn up -d recyclarr listsync
```

Valide o Recyclarr:

```bash
docker exec recyclarr recyclarr sync
```

Deve criar profiles "UHD Bluray + WEB" no Radarr e "WEB-2160p" no Sonarr.

Configure o ListSync em http://SEU_IP:3222 (conectar watchlists do Letterboxd, IMDB, AniList, etc).

### 8. Bazarr - Providers de legenda

No Bazarr, ative providers para melhor cobertura de legendas PT-BR:

- **OpenSubtitles.com** (criar conta gratuita em opensubtitles.com)
- **Legendas.net**
- **Podnapisi**
- **Subf2m**
- **Gestdown**

Recomendacoes:
- Minimum score para series: **70** (padrao 90 e muito restritivo)
- Minimum score para filmes: **70**
- Ativar **subsync** para sincronizar timing automaticamente

## Notas Windows

- **PUID/PGID**: podem ser ignorados no Windows (Docker Desktop gerencia permissoes). Deixe os valores padrao.
- **Paths**: use formato Windows (`D:\Downloads`) ou formato Unix (`//d/Downloads`)
- **Performance**: volumes bind-mounted do Windows para WSL2 sao mais lentos. Para melhor performance, armazene os dados dentro do WSL2.

## Portas

| Servico      | Porta |
|--------------|-------|
| qBittorrent  | 5080  |
| Radarr       | 7878  |
| Sonarr       | 8989  |
| Prowlarr     | 9696  |
| Jellyfin     | 8096  |
| Jellyseerr   | 5055  |
| Bazarr       | 6767  |
| ListSync     | 3222  |
| Recommendarr | 3000  |
