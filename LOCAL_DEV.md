# Snow Resorts — desenvolvimento local

Guia passo a passo para subir **toda a aplicação** no Mac: infra Docker, 5 microserviços Java, gateway nginx e app mobile no simulador iOS.

Custo: **$0** (tudo roda na sua máquina).

---

## Visão geral

```
Simulador iOS / dispositivo
    │
    ├─► Metro (JS)          :8086
    │
    └─► Gateway nginx       :8080
            ├─► auth-service      :8081
            ├─► user-service      :8082
            ├─► resort-service    :8083
            ├─► location-service  :8084
            └─► activity-service  :8085
                    │
            Docker: Postgres+PostGIS, Redis, MinIO, Mailpit
```

| Porta | Serviço |
|-------|---------|
| `5432` ou `5433` | Postgres (Docker) — ver nota abaixo |
| `6379` | Redis |
| `9000` / `9001` | MinIO (S3 local) / console |
| `1025` / `8025` | Mailpit (SMTP / UI) |
| `8080` | Gateway nginx + Swagger |
| `8081`–`8085` | Microserviços Java |
| `8086` | Metro bundler (app mobile) |

---

## Pré-requisitos

Instale e deixe rodando:

| Ferramenta | Versão / nota |
|------------|----------------|
| **Docker Desktop** | Em execução |
| **Java 25** | `JAVA_HOME` configurado (`/usr/libexec/java_home -v 25`) |
| **Node.js 26** | Ex.: `~/.local/node26` no `PATH` |
| **Xcode** | Com runtime do **simulador iOS** (Xcode → Settings → Platforms) |
| **CocoaPods** | `pod --version` |
| **GitHub PAT** | Para resolver libs no Maven (ver abaixo) |

O app mobile **não roda no Expo Go** — usa Mapbox, MMKV e New Architecture. É necessário **dev client** (`expo-dev-client`).

---

## Configuração única (primeira vez)

### 1. Maven — GitHub Packages

Copie ou mescle em `~/.m2/settings.xml` (veja `snow-resorts-shared/settings.xml.example`):

```xml
<server>
  <id>github</id>
  <username>SEU_USUARIO_GITHUB</username>
  <password>${env.GITHUB_TOKEN}</password>
</server>
```

Exporte o token antes de buildar os serviços:

```bash
export GITHUB_TOKEN=ghp_xxxxxxxx
```

Alternativa local (sem GitHub Packages): instale as libs no `~/.m2`:

```bash
cd snow-resorts-shared
./mvnw install
```

### 2. Biblioteca compartilhada (local)

```bash
cd snow-resorts-shared
./mvnw install
```

### 3. Mobile — dependências e tokens

```bash
cd snow-resorts-mobile
npm install
```

Crie **`snow-resorts-mobile/.env`** (não versionado):

```env
# Token secreto Mapbox (escopo Downloads:Read) — build nativo (pod install).
RNMAPBOX_MAPS_DOWNLOAD_TOKEN=sk.seu_token_aqui

# Token público Mapbox — runtime do mapa.
EXPO_PUBLIC_MAPBOX_TOKEN=pk.seu_token_aqui
```

Crie **`snow-resorts-mobile/.env.local`** (não versionado):

```env
# Simulador iOS: deep link usa localhost (evita timeout com IP da LAN).
REACT_NATIVE_PACKAGER_HOSTNAME=localhost

# Metro na 8086 (8081–8085 são os microserviços Java).
RCT_METRO_PORT=8086
```

### 4. Mobile — build nativo iOS (primeira vez)

```bash
cd snow-resorts-mobile
npx expo prebuild --platform ios --clean
npm run ios
```

Na primeira build, confirme a licença do Xcode se pedido:

```bash
sudo xcodebuild -license accept
```

O arquivo `ios/.xcode.env` fixa `RCT_METRO_PORT=8086` (versionado). O `ios/.xcode.env.local` só precisa apontar para o Node 26, se o `command -v node` do sistema não for o Node 26:

```env
export NODE_BINARY=/Users/SEU_USUARIO/.local/node26/bin/node
```

**Importante:** não use `--no-bundler` no `ios:sim` — o Expo CLI ignora `RCT_METRO_PORT` nesse modo e sempre abre o app na porta **8081** (que é o auth-service). O script `npm run ios:sim` passa `--port 8086` e reutiliza o Metro se ele já estiver rodando em outro terminal.

---

## Rotina diária — subir tudo

### Passo 1 — Infra Docker

```bash
cd snow-resorts-infra
make dev
```

Isso sobe Postgres+PostGIS, Redis, MinIO, Mailpit, gateway nginx e executa o seed de dados demo.

Se a saída indicar Postgres na **5433** (porque algo já usa a 5432):

```bash
export POSTGRES_PORT=5433
```

O valor também fica em `snow-resorts-infra/.env` após `make up`.

**Verificar:**

```bash
docker compose -f docker/docker-compose.yml ps
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/swagger/
```

### Passo 2 — Cinco microserviços Java

Abra **5 terminais** (ou use um gerenciador de processos). Em cada um:

```bash
# Terminal 1 — auth (:8081)
cd snow-resorts-auth-service
./mvnw spring-boot:run

# Terminal 2 — user (:8082)
cd snow-resorts-user-service
./mvnw spring-boot:run

# Terminal 3 — resort (:8083)
cd snow-resorts-resort-service
./mvnw spring-boot:run

# Terminal 4 — location (:8084)
cd snow-resorts-location-service
./mvnw spring-boot:run

# Terminal 5 — activity (:8085)
cd snow-resorts-activity-service
./mvnw spring-boot:run
```

Cada serviço usa o profile **`local`** e conecta no Postgres do Docker.

Se `POSTGRES_PORT=5433`, exporte **antes** de iniciar os serviços no mesmo terminal.

**Reinicie o resort-service** (`snow-resorts-resort-service`) após puxar migrações Flyway novas (ex.: V2–V4 com seeds de resorts). O Spring Boot aplica as migrações só na subida; sem restart, o catálogo no app pode ficar desatualizado mesmo com `make seed`.

**Após todos subirem**, reexecute o seed se resorts/perfil demo não aparecerem:

```bash
cd snow-resorts-infra z
make seed
```

### Passo 3 — App mobile no simulador

**Opção A — dois terminais (recomendado)**

Terminal A — Metro:

```bash
cd snow-resorts-mobile
npm start
```

Terminal B — compilar/instalar no simulador (reutiliza o Metro do terminal A se já estiver na 8086):

```bash
cd snow-resorts-mobile
npm run ios:sim
```

Se o log mostrar `Waiting on http://localhost:8086`, está correto. Se aparecer **8081**, recompile com `npm run ios:sim` após `pod install` no diretório `ios/`.

No simulador: abra **Snow Resorts** → toque no servidor com bolinha verde (`http://localhost:8086`) ou em **Open**.

**Opção B — um comando (Metro + build)**

```bash
cd snow-resorts-mobile
npm run ios
```

Se aparecer timeout no `openurl` no final, ignore — o app já foi instalado; abra manualmente no simulador.

**Após mudar `RCT_METRO_PORT` ou `ios/.xcode.env.local`**, recompile:

```bash
npm run ios:sim
```

---

## Login de teste

| Campo | Valor |
|-------|--------|
| Email | `demo@snow-resorts.com` |
| Senha | `Password123!` |

---

## URLs úteis

| URL | Descrição |
|-----|-----------|
| http://localhost:8080/snow-resort-service/v1 | API (usada pelo app) |
| ws://localhost:8080/ws | WebSocket (localização em tempo real) |
| http://localhost:8080/swagger/ | Swagger unificado (5 serviços) |
| http://localhost:9001 | MinIO console (`minioadmin` / `minioadmin`) |
| http://localhost:8025 | Mailpit (e-mails de dev) |
| http://localhost:8086 | Metro bundler |

---

## Parar tudo

```bash
# Infra Docker
cd snow-resorts-infra
make down

# Microserviços Java: Ctrl+C em cada terminal

# Metro: Ctrl+C no terminal do npm start
```

Reset completo do banco Docker (apaga volumes):

```bash
cd snow-resorts-infra
make clean-data
make dev
```

---

## Problemas comuns

### `ports are not available: 5432`

Outro Postgres (ou serviço) usa a 5432. O `make dev` escolhe **5433** automaticamente. Exporte:

```bash
export POSTGRES_PORT=5433
```

antes de subir os microserviços Java.

### `Failed to load app from http://localhost:8081`

O binário iOS foi compilado com a porta padrão **8081**, mas o Metro está na **8086**.

1. Confirme `RCT_METRO_PORT=8086` em `.env.local` e `ios/.xcode.env.local`.
2. Recompile: `npm run ios:sim`.
3. Atalho sem rebuild: no dev client, **Enter URL manually** → `http://localhost:8086`.

### `simctl openurl` timeout (code 60)

O Expo tentou abrir deep link com IP da LAN (`192.168.x.x`). Use `REACT_NATIVE_PACKAGER_HOSTNAME=localhost` no `.env.local` e abra o app manualmente no simulador.

### `No script URL provided`

Metro não está rodando ou o dev client não conectou. Suba `npm start` e toque no servidor `localhost:8086` no app.

### Mapbox / `pod install` falha

Confirme `RNMAPBOX_MAPS_DOWNLOAD_TOKEN` no `.env` antes de `npx expo prebuild`.

### `geometry does not exist` (resort-service)

PostGIS não carregou. Reset da infra:

```bash
cd snow-resorts-infra
make clean-data && make dev
```

### Simulador não encontrado

Instale o runtime iOS no Xcode ou baixe:

```bash
xcodebuild -downloadPlatform iOS
```

---

## Ordem resumida (checklist)

- [ ] Docker Desktop rodando
- [ ] `cd snow-resorts-shared && ./mvnw install` (primeira vez ou após mudanças na lib)
- [ ] `cd snow-resorts-infra && make dev`
- [ ] `export POSTGRES_PORT=5433` (se o make avisar)
- [ ] Subir auth, user, resort, location, activity (`./mvnw spring-boot:run`)
- [ ] `make seed` (se necessário)
- [ ] `cd snow-resorts-mobile && npm start` (terminal 1)
- [ ] `npm run ios:sim` ou `npm run ios` (terminal 2)
- [ ] Login: `demo@snow-resorts.com` / `Password123!`
