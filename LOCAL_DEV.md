# Snow Resorts — desenvolvimento local

Guia passo a passo para subir **toda a aplicação** no Mac: infra Docker, 5 microserviços Java, gateway nginx e app mobile no simulador iOS.

**Arquitetura da plataforma:** [ARCHITECTURE.md](./ARCHITECTURE.md)

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


| Porta            | Serviço                             |
| ---------------- | ----------------------------------- |
| `5432` ou `5433` | Postgres (Docker) — ver nota abaixo |
| `6379`           | Redis                               |
| `9000` / `9001`  | MinIO (S3 local) / console          |
| `1025` / `8025`  | Mailpit (SMTP / UI)                 |
| `8080`           | Gateway nginx + Swagger             |
| `8081`–`8085`    | Microserviços Java                  |
| `8086`           | Metro bundler (app mobile)          |


---



## Pré-requisitos

Instale e deixe rodando:


| Ferramenta         | Versão / nota                                                   |
| ------------------ | --------------------------------------------------------------- |
| **Docker Desktop** | Em execução                                                     |
| **Java 25**        | `JAVA_HOME` configurado (`/usr/libexec/java_home -v 25`)        |
| **Node.js 26**     | Ex.: `~/.local/node26` no `PATH`                                |
| **Xcode**          | Com runtime do **simulador iOS** (Xcode → Settings → Platforms) |
| **CocoaPods**      | `pod --version`                                                 |
| **GitHub PAT**     | Para resolver libs no Maven (ver abaixo)                        |


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

Após puxar mudanças no `package.json`, rode `npm install` de novo.

Crie `**snow-resorts-mobile/.env**` (não versionado):

```env
# Token secreto Mapbox (escopo Downloads:Read) — build nativo (pod install).
RNMAPBOX_MAPS_DOWNLOAD_TOKEN=sk.seu_token_aqui

# Token público Mapbox — runtime do mapa.
EXPO_PUBLIC_MAPBOX_TOKEN=pk.seu_token_aqui
```

Crie `**snow-resorts-mobile/.env.local**` (não versionado):

**Simulador iOS:**

```env
REACT_NATIVE_PACKAGER_HOSTNAME=localhost
RCT_METRO_PORT=8086
```

**Celular físico (mesmo Wi‑Fi que o Mac):**

```env
REACT_NATIVE_PACKAGER_HOSTNAME=192.168.3.18
RCT_METRO_PORT=8086
EXPO_PUBLIC_API_BASE_URL=http://192.168.3.18:8080/snow-resort-service/v1
EXPO_PUBLIC_WS_BASE_URL=ws://192.168.3.18:8080/ws
```

Substitua `192.168.3.18` pelo IP do Mac (`ipconfig getifaddr en0`).

Para **Esqueci minha senha** no celular, crie `snow-resorts-auth-service/.env.local` (copie de `[.env.local.example](snow-resorts-auth-service/.env.local.example)`) com o **mesmo IP** do mobile. O profile `local` carrega esse arquivo automaticamente ao rodar `./mvnw spring-boot:run` — não precisa `export` manual.

Para **foto de perfil** no celular, o `user-service` (profile `local`) já usa o mesmo IP em MinIO (`AVATAR_PUBLIC_BASE_URL` / `S3_ENDPOINT`, default `192.168.3.18:9000`). Se o IP do Mac mudar, exporte esses envs ou ajuste `application-local.yml` e reinicie o user-service.

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

### Estrutura do código mobile


| Pasta             | Conteúdo                                                                                              |
| ----------------- | ----------------------------------------------------------------------------------------------------- |
| `app/`            | Rotas Expo Router (telas = `export default function …Screen()`)                                       |
| `src/theme/`      | Tokens — `colors.ts` (paleta), `interactive.ts` (classes NativeWind), `layout.ts` (dimensões do mapa) |
| `src/components/` | UI reutilizável (`Button`, `Segmented`, `auth/*`, `map/*`)                                            |
| `src/hooks/`      | React Query + GPS/descida                                                                             |
| `src/api/`        | Cliente axios + contratos tipados                                                                     |
| `src/dev/`        | Mock GPS stub + types (`mockGps.stub.ts` fallback)                                                    |


Telas de auth usam `AuthScreenLayout`; demais telas usam `SafeAreaView` / `ScrollView` diretamente. Superfícies e inputs devem importar classes de `@/theme/interactive`.

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

Cada serviço usa o profile `**local**` e conecta no Postgres do Docker.

Se `POSTGRES_PORT=5433`, exporte **antes** de iniciar os serviços no mesmo terminal.

**Reinicie o resort-service** (`snow-resorts-resort-service`) após puxar migrações Flyway novas (ex.: V2–V4 com seeds de resorts). O Spring Boot aplica as migrações só na subida; sem restart, o catálogo no app pode ficar desatualizado mesmo com `make seed`.

**Reinicie o activity-service** após migrações novas (ex.: `V3__drop_run_tracks_s3.sql`).

**Após todos subirem**, reexecute o seed se resorts/perfil demo não aparecerem:

```bash
cd snow-resorts-infra
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

**Após mudar** `RCT_METRO_PORT` **ou** `ios/.xcode.env.local`, recompile:

```bash
npm run ios:sim
```



### GPS simulado — descida no Valle Nevado (opcional)

Para testar **mapa + descida** sem GPS real no simulador, use o mock de Valle Nevado (estacionamento → Bajo Cero → caminhada até a Diablada → Cuando).

**Como funciona**


| O quê                          | Onde                                                     | Git                         |
| ------------------------------ | -------------------------------------------------------- | --------------------------- |
| Simulador (padrão)             | `snow-resorts-mobile/dev-local.mock-gps.example/`        | commitado                   |
| Override local (opcional)      | `snow-resorts-mobile/dev-local/mock-gps/`                | **ignorado** (`.gitignore`) |
| Fallback                       | `src/dev/mockGps.stub.ts` (GPS real)                     | commitado                   |


O Metro resolve `@local/mock-gps` nesta ordem: `dev-local/mock-gps/` (se existir) → template commitado → stub.

> **Atenção:** os comandos abaixo são só para copiar do bloco de código — **não** cole símbolos da saída do terminal (`√`, `›`, etc.). Se aparecer `command not found: √`, foi isso.
>
> Rode sempre dentro de `snow-resorts-mobile/` (não na pasta raiz `snow-resorts/`).

**Terminal 1 — Metro com mock**

```bash
cd snow-resorts-mobile
npm run start:mock-gps
```

Por padrão o mock vale **só no simulador/emulador**. Celular físico no mesmo Metro usa GPS real. Para forçar mock no device: `EXPO_PUBLIC_MOCK_ON_DEVICE=true` (junto com `start:mock-gps`).

**Terminal 2 — simulador iOS**

```bash
cd snow-resorts-mobile
npm run ios:sim
```

No celular (mesmo Metro / Wi‑Fi do Mac): abra o app normalmente — sem badge de GPS simulado.
**No app**

- Badge no mapa: `GPS simulado · Valle Nevado · [fase]` (ex.: *Pista Sol 2*)
- Ponto azul se move pela rota (~10 min, depois reinicia no estacionamento)
- **Reiniciar rota** — volta ao estacionamento
- **Iniciar descida** — grava velocidade, altitude e distância como GPS real
- Valle Nevado é selecionado automaticamente

**Variáveis (opcionais)**


| Variável                         | Efeito                                                                               |
| -------------------------------- | ------------------------------------------------------------------------------------ |
| `EXPO_PUBLIC_MOCK_LOCATION=true` | Liga o mock no **simulador** (`start:mock-gps` já define)                             |
| `EXPO_PUBLIC_MOCK_ON_DEVICE=true` | Também liga mock no **celular físico** (opcional; testes de background com mock)    |

> `.env.local` **tem prioridade sobre** `.env`**.** Se o mock continuar ligado com `false` no `.env`, verifique se `.env.local` não define `EXPO_PUBLIC_MOCK_LOCATION=true`. Depois de mudar, reinicie o Metro com `npx expo start --clear --port 8086`.

**Remover / desativar o mock**

1. Use `EXPO_PUBLIC_MOCK_LOCATION=false` (ou remova a variável) e reinicie o Metro
2. (Opcional) apague `dev-local/mock-gps/` se tiver criado uma cópia local para override

Mantenha `src/dev/mockGps.stub.ts` e `src/dev/mockGps.types.ts` — fallback commitado.

#### Mock GPS em segundo plano

Com mock ligado, a rota avança em background pelo mesmo `expo-task-manager` da localização de grupo (`advanceTo` por hora do sistema). Precisa de **dev client** + permissão **Sempre** (e `MOCK_ON_DEVICE` no celular físico). Cadência e overlay: ver [Localização em segundo plano](#localização-em-segundo-plano-grupo-ativo).

#### Mock + grupo cross-device (celular publica, simulador recebe)

| Dispositivo | Papel | Config |
| ----------- | ----- | ------ |
| **Celular físico** | publicador | GPS real **ou** mock com `MOCK_ON_DEVICE=true`; no mesmo grupo; perfil **Compartilhar localização → Amigos**; **Sempre**; dev client |
| **Simulador iOS** | receptor | mesmo Metro (`start:mock-gps`); mesmo grupo |

Bloqueie o celular 30–60 s e no simulador confira `amigo` no overlay / Redis (`HGETALL location:group:<UUID>`). O simulador não é publicador BG confiável.

### Velocidade com GPS real (celular na pista)

Com `EXPO_PUBLIC_MOCK_LOCATION=false`, a descida usa o chip GNSS do aparelho (`Accuracy.BestForNavigation`, updates a cada **1 s**). A velocidade segue esta ordem:

1. **`coords.speed` do SO** (m/s) — quando o chip envia valor válido (mais preciso).
2. **Derivação em janela** — soma das distâncias dos últimos 2–3 fixes ÷ tempo total, quando `speed` não vem.
3. **Fallback ponto a ponto** — só se a janela ainda não tiver dados suficientes.

O HUD aplica uma média curta (3 amostras) só para exibição; os pontos gravados usam velocidade filtrada (Kalman). Fixes com `horizontalAccuracy` pior que **35 m** são descartados; velocidade do chip é ignorada se accuracy > 20 m. Teto de spike ~**150 km/h**; saltos de posição escalam com Δt para não cortar descidas rápidas com o celular bloqueado. Inclinação: distância mínima ~**3 m**, EMA (α=0.25) no pico/média para um spike GNSS não virar 85°; teto ~**75°**. Em background, todos os fixes do wake entram na descida. **Altitude/desnível em ambiente interno** (escadas) continua limitado pelo GNSS do celular. Precisão melhor ao ar livre na pista.

---



## Login de teste


| Campo | Valor                   |
| ----- | ----------------------- |
| Email | `demo@snow-resorts.com` |
| Senha | `Password123!`          |


---



## Recuperar senha (Mailpit + redirect)

E-mails de recuperação **não** vão para Gmail/Mail do celular — ficam no **Mailpit** (SMTP local, custo **$0**). O auth-service (profile `local`) envia via `localhost:1025`; sem Mailpit/SMTP, o token cai só no log do auth-service.


| Dispositivo                  | URL do Mailpit                                              |
| ---------------------------- | ----------------------------------------------------------- |
| Simulador iOS (Safari)       | [http://localhost:8025](http://localhost:8025)              |
| Celular físico (mesmo Wi‑Fi) | `http://<IP_DO_MAC>:8025` (ex.: `http://192.168.3.18:8025`) |


**Link no e-mail (HTML, clicável):** `http://localhost:8080/reset-password?token=...` (simulador) ou `http://<IP_DO_MAC>:8080/reset-password?token=...` (celular, via `.env.local` do auth-service).

**Celular físico:** crie `snow-resorts-auth-service/.env.local` com `PASSWORD_RESET_BASE_URL=http://<IP_DO_MAC>:8080/reset-password` (mesmo IP do mobile). Carregado automaticamente no `./mvnw spring-boot:run`. Mailpit: `http://<IP_DO_MAC>:8025`.

**Fluxo:** app → **Esqueci minha senha** → `demo@snow-resorts.com` → Mailpit **no mesmo dispositivo** do app → toque **Reset your password** → redirect HTTP → app em **Redefinir senha** (token preenchido).

**Não funciona:** Mailpit no Chrome do Mac (não abre o app no simulador/celular); colar `snowresorts://...` na barra do navegador; link `localhost` no celular físico.

**Reiniciar após mudanças:** auth-service (`application-local.yml`); nginx — `cd snow-resorts-infra/docker && docker compose up -d nginx` (página estática nova).

---



## URLs úteis


| URL                                                                                          | Descrição                                   |
| -------------------------------------------------------------------------------------------- | ------------------------------------------- |
| [http://localhost:8080/snow-resort-service/v1](http://localhost:8080/snow-resort-service/v1) | API (usada pelo app)                        |
| ws://localhost:8080/ws                                                                       | WebSocket (localização em tempo real)       |
| [http://localhost:8080/swagger/](http://localhost:8080/swagger/)                             | Swagger unificado (5 serviços)              |
| [http://localhost:9001](http://localhost:9001)                                               | MinIO console (`minioadmin` / `minioadmin`) |
| [http://localhost:8025](http://localhost:8025)                                               | Mailpit (e-mails de dev)                    |
| [http://localhost:8080/reset-password?token=…](http://localhost:8080/reset-password)         | Redirect local → app (`snowresorts://`)     |
| [http://localhost:8086](http://localhost:8086)                                               | Metro bundler                               |


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

### `PrivacyInfo.xcprivacy` couldn't be opened (Xcode)

O projeto iOS referencia `ios/SnowResorts/PrivacyInfo.xcprivacy` (manifesto de privacidade exigido pela Apple). A pasta `ios/` não está no git; após `expo prebuild --clean` ou edições manuais o arquivo pode sumir.

```bash
cd snow-resorts-mobile
npm run ensure:ios-privacy
cd ios && pod install && cd ..
```

Depois faça **Product → Clean Build Folder** no Xcode e compile de novo. O template versionado fica em `support/ios/PrivacyInfo.xcprivacy`.

### `The host [location_service] is not valid` (location-service / WebSocket)

O nginx, no bloco `/ws`, enviava `Host: location_service` (nome do upstream com `_`). O Tomcat 10+ rejeita isso. Corrigido em `snow-resorts-infra/docker/nginx/nginx.conf` — recarregue o gateway:

```bash
cd snow-resorts-infra
docker compose -f docker/docker-compose.yml restart nginx
```

### `STOMP off` no mapa (localização do amigo parada)

1. Confirme que você está no grupo (aba Amigos → Grupos) e que o perfil compartilha localização com amigos.
2. Reinicie o **location-service** após mudanças no backend.
3. Recarregue o app (Metro) — o cliente STOMP precisa negociar o subprotocolo `v12.stomp` (corrigido em `locationStompClient.ts`; não use `webSocketFactory` sem os protocolos).
4. No overlay dev: `in` = último push STOMP **recebido** do amigo; `try`/`pub` = publish **deste aparelho**; `euVia=stomp|rest` = canal da **sua** última tentativa (não do amigo); `bgWake` = último wake/keep-alive persistido (publisher); `amigo` / `srv eu|amigo` = idade ao vivo do `recordedAt`. Se `amigo` e `srv amigo` estão baixos, o amigo está chegando — mesmo que `euVia=rest`. No Network, `GET .../positions` é bootstrap pontual (não o feed live); `POST .../position` é publish REST.

### Localização em segundo plano (grupo ativo)

Com membership no grupo e localização compartilhada no perfil, o app continua publicando GPS com a tela bloqueada ou em outro app (celular no bolso).

**Configuração (app):**

- Foreground: `watchPositionAsync` a cada **5 s** (`Accuracy.High`) via **STOMP**.
- Background: `expo-task-manager` com `pausesUpdatesAutomatically: false`, `activityType: Fitness`, `distanceInterval: 0`, **`deferredUpdatesInterval` ~15 s** (iOS ignora `timeInterval`).
- Publicação: **STOMP** enquanto a sessão estiver saudável no keep-alive (~30 s Apple); depois **REST** no farewell + em cada wake de CLLocation (`preferRest`, com `beginBackgroundTask` no wake).
- Overlay no **publisher** (ao desbloquear): `bgWake Xs ok|fail:reason` — se `fail:ensure_sharing_failed` / `no_token` / `no_group`, o wake não publicou.
- **Descida ao vivo:** usa a **mesma** task de background do grupo (sem segunda sessão nativa). Com o app aberto grava ~1 s; com a tela bloqueada recebe os wakes da task (~5 s se só descida, ~15 s se o grupo também está ativo) e continua as métricas. Ao sair do grupo com uma descida ativa, a task **não** é parada.

**Requisitos:**

- **Dev client** — não funciona no Expo Go; após mudanças nativas (`expo-task-manager`), rebuild:
  ```bash
  cd snow-resorts-mobile
  npm run ios   # ou npm run android
  ```
- **iOS:** com o grupo criado/entrado e localização compartilhada no perfil, conceda localização **Sempre**. Se o overlay mostrar `bgWake fail:no_always_permission`, abra Ajustes → Snow Resorts → Localização → **Sempre**. Sem Always, o app até recebe o amigo via STOMP, mas **não publica** de forma confiável após o lock. `task_not_defined` = Reload completo (não só Fast Refresh); a task deve estar registrada em `index.js` **e** `app/_layout.tsx`, e `ensureGroupLocationTaskDefined()` roda no start. Teste com **celular físico + andar**.
- **Android:** notificação persistente enquanto estiver no grupo com compartilhamento ligado.
- **Simulador iOS** não é publicador BG confiável — use celular físico como publicador.

**Teste:** entre no grupo no celular (Always + perfil compartilha localização), bloqueie **>40 s** andando, confira no Redis/`amigo` no sim. Unlock no celular: overlay `bgWake … ok:wake_fix` (ou `ok:keep_alive_farewell`). Com mock só no sim (`start:mock-gps` sem `MOCK_ON_DEVICE`), o celular usa GPS real nesse fluxo.

### Posições em tempo real (Redis)

O **location-service** guarda a posição atual de cada membro **somente em Redis** (`location:group:{groupId}`, HASH com TTL de **2h** renovado a cada upsert). Ao sair do grupo ou definir **Compartilhar localização → Ninguém**, o app chama `DELETE .../position` e o peer recebe um clear STOMP. Membership continua no Postgres enquanto o grupo existir (um grupo por usuário; expira em 12h).

Verificar no Redis após publicar posições:

```bash
docker exec -it snow-redis redis-cli KEYS 'location:group:*'
docker exec -it snow-redis redis-cli TTL 'location:group:<UUID_DO_GRUPO>'
docker exec -it snow-redis redis-cli HGETALL 'location:group:<UUID_DO_GRUPO>'
```

Config em `snow-resorts-location-service/src/main/resources/application.yml`:

```yaml
location:
  positions:
    retention-hours: 24
```

### `geography` / `geometry does not exist` (resort-service)

PostGIS está instalado, mas o `currentSchema=resorts` na JDBC deixa o `search_path` só em `resorts`, sem `public` (onde ficam os tipos/funções PostGIS). Reinicie o resort-service após atualizar o código. Se persistir, reset da infra:

```bash
cd snow-resorts-infra
make clean-data && make dev
```



### Simulador não encontrado

Instale o runtime iOS no Xcode ou baixe:

```bash
xcodebuild -downloadPlatform iOS
```



### Link de recuperar senha não abre o app

1. Abra o Mailpit **no simulador/celular**, não no Mac.
2. Toque o link **HTTP** do e-mail (`http://…:8080/reset-password?token=…`), não cole `snowresorts://` manualmente.
3. Celular físico: exporte `PASSWORD_RESET_BASE_URL` com o **IP do Mac** antes de subir o auth-service.
4. Confirme nginx com a página de redirect: `curl -s "http://localhost:8080/reset-password?token=test" | head -3`
5. Reinicie o auth-service após mudar config de e-mail.

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
- [ ] (Opcional) GPS simulado — terminal 1: `cd snow-resorts-mobile && npm run start:mock-gps`
- [ ] (Opcional) GPS simulado — terminal 2: `cd snow-resorts-mobile && npm run ios:sim`
- [ ] Login: `demo@snow-resorts.com` / `Password123!`
- [ ] (Opcional) Recuperar senha — Mailpit no mesmo device; link HTTP → redirect → app
- [ ] (Celular físico) `export PASSWORD_RESET_BASE_URL=http://<IP_MAC>:8080/reset-password` antes do auth-service
