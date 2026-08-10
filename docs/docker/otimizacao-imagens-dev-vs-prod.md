# Otimização de Imagens + Estratégia Dev vs Prod

> **Objetivo:** Escrever Dockerfiles eficientes, entender o impacto das camadas no cache, e estruturar seu projeto para ter **hot reload no desenvolvimento** e **imagens imutáveis/seguras na produção** — tudo com Docker Compose.
> 

!!! warning "Este capítulo é referência conceitual, não uma continuação direta do projeto anterior"
    Os exemplos abaixo (Postgres como `db`, healthcheck em `/health`, scripts `npm run dev`/`npm run build`, `nodemon`, `Dockerfile.dev`) formam um **projeto de referência separado**, montado só para ilustrar os padrões de otimização e a estratégia dev/prod.

    Eles **não** se encaixam diretamente no projeto `compose-step-by-step` construído no capítulo anterior — lá o serviço era `cache` (Redis) e o `index.js` usava só o módulo `http`, sem rota `/health` nem scripts de build. Se você copiar esses trechos por cima daquele projeto, o `healthcheck` vai falhar (não existe `/health`) e os comandos `npm run dev`/`build` vão dar erro (não existem no `package.json`).

    Use este capítulo para **entender os padrões** — camadas, cache, multi-stage, `.dockerignore`, dev vs prod — e aplicá-los ao seu próprio projeto, adaptando os nomes de serviço, variáveis e scripts conforme necessário.

---

### 🧱 1. Fundamentos: Como Funcionam as Camadas (Layers)

Cada instrução no `Dockerfile` cria uma camada `read-only`. O Docker armazena essas camadas em cache e **reutiliza** se a instrução e seus arquivos de contexto não mudaram.

```docker
FROM node:20-alpine        # Camada 1: imagem base
WORKDIR /app               # Camada 2: cria diretório
COPY package*.json ./      # Camada 3: copia manifestos
RUN npm install            # Camada 4: instala dependências ← CACHEABLE
COPY . .                   # Camada 5: copia código fonte ← MUDA SEMPRE
EXPOSE 3000                # Camada 6: metadata
CMD ["npm", "start"]       # Camada 7: comando de entrada
```

🔍 **Regra de Ouro do Cache:**

O cache é invalidado **a partir da primeira camada que muda**. Se você alterar `index.js` e fizer `COPY . .` **antes** do `RUN npm install`, o Docker invalida o cache do `npm install` e reinstala tudo (lento!).

### ❌ Ordem Ineficiente (cache sempre invalidado)

```docker
COPY . .              # Muda a cada edição de código
RUN npm install       # ← Cache invalidado → reinstala tudo
```

### ✅ Ordem Otimizada (cache inteligente)

```docker
COPY package*.json ./ # Muda só quando dependências mudam
RUN npm install       # ← Cache REUTILIZADO se package.json não mudar
COPY . .              # Muda frequentemente, mas fica DEPOIS do install
```

🧪 **Teste prático:**

```bash
# Build inicial (lento)
docker compose build app

# Altere apenas o código
echo "// comment" >> app/index.js

# Build novamente (rápido)
docker compose build app
# → Procure por "CACHED" no output do npm install
```

---

### 🗑️ 2. Use `.dockerignore` para Contexto Limpo

O Docker envia **toda a pasta** para o daemon antes de construir. Arquivos desnecessários aumentam o tempo de transferência e podem vazar segredos.

### Crie `app/.dockerignore`:

```
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
*.log
.DS_Store
coverage/
dist/
*.md
```

🔍 **Por que ignorar `node_modules`?**

- Se você rodou `npm install` no host, a pasta pode ser incompatível com Alpine (binários compilados para glibc vs musl).
- O `RUN npm install` no Dockerfile já instala as dependências corretas para o ambiente do container.

---

### 🏗️ 3. Multi-Stage Builds: Imagens Mínimas para Produção

Para projetos que compilam código (TypeScript, Go, Java) ou instalam dependências de desenvolvimento, use estágios múltiplos para separar **build** de **runtime**.

### `app/Dockerfile` (versão multi-stage para produção)

```docker
# ─────────────────────────────────────
# ESTÁGIO 1: BUILD (compila/transpila)
# ─────────────────────────────────────
FROM node:20-alpine AS builder

WORKDIR /app

# Copia manifestos primeiro (cache de dependências)
COPY package*.json ./
RUN npm install  # Instala tudo (dev + prod)

# Copia código fonte e compila
COPY . .
RUN npm run build  # Gera pasta dist/ com código transpilado

# ─────────────────────────────────────
# ESTÁGIO 2: RUNTIME (imagem final mínima)
# ─────────────────────────────────────
FROM node:20-alpine

# Metadados para auditoria
LABEL maintainer="seu-email@example.com"
LABEL version="1.0"

WORKDIR /app

# Copia apenas manifestos para instalar só produção
COPY package*.json ./
RUN npm install --only=production && npm cache clean --force

# Copia apenas o código compilado do estágio anterior
COPY --from=builder /app/dist ./dist

# Usuário não-root (segurança básica)
USER node

EXPOSE 3000

# Comando de entrada aponta para o código compilado
CMD ["node", "dist/index.js"]
```

🔍 **Resultado:**

| Métrica | Dockerfile Simples | Multi-Stage |
| --- | --- | --- |
| Tamanho da imagem | ~300 MB | ~50 MB |
| Contém código fonte? | ✅ Sim | ❌ Não |
| Contém devDependencies? | ✅ Sim | ❌ Não |
| Contém ferramentas de build? | ✅ Sim | ❌ Não |
| Superfície de ataque | Maior | Mínima |

---

### 🔄 4. Estratégia Dev vs Prod com Docker Compose

> **Filosofia:** Um `docker-compose.yml` base define a arquitetura. Arquivos de override adaptam o comportamento por ambiente.
> 

### 📁 Estrutura de Arquivos Recomendada

```
projeto/
├── docker-compose.yml          # Base: serviços, redes, volumes (comum)
├── docker-compose.dev.yml      # Override: hot reload, debug, volumes
├── docker-compose.prod.yml     # Override: imagem imutável, segurança, limits
├── .env.example                # Template de variáveis (commitar)
├── .env                        # Valores reais (NÃO commitar)
├── app/
│   ├── Dockerfile              # Multi-stage para prod
│   ├── Dockerfile.dev          # Simples, com nodemon para dev
│   └── .dockerignore
└── README.md
```

---

### 📄 `docker-compose.yml` (Base — Comum a Todos)

```yaml
services:
  app:
    build:
      context: ./app
      dockerfile: Dockerfile  # Default: usa o de produção
    ports:
      - "${APP_PORT:-3000}:3000"
    environment:
      - DB_HOST=db
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
      - NODE_ENV=${NODE_ENV:-production}
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-net
    healthcheck:
      test: ["CMD", "node", "-e", "require('http').get('http://localhost:3000/health', r => process.exit(r.statusCode===200?0:1))"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-net

volumes:
  pg_data:
networks:
  app-net:
    driver: bridge
```

---

### 🛠️ `docker-compose.dev.yml` (Desenvolvimento)

```yaml
services:
  app:
    build:
      context: ./app
      dockerfile: Dockerfile.dev  # Usa imagem com nodemon
    volumes:
      - ./app:/app:cached         # Bind mount para hot reload
      - /app/node_modules         # Preserva node_modules do container
    environment:
      - NODE_ENV=development
      - WATCHPACK_POLLING=true    # Necessário em WSL2/macOS
    command: ["npm", "run", "dev"]  # Override: roda nodemon
    healthcheck:
      disable: true  # Desabilita em dev para iteração rápida

  # Opcional: serviço de debug com inspector
  debug:
    image: node:20-alpine
    volumes:
      - ./app:/app
    working_dir: /app
    command: ["sh", "-c", "npm install && node --inspect=0.0.0.0:9229 index.js"]
    ports:
      - "9229:9229"  # Node inspector
    profiles: ["debug"]  # Só sobe com --profile debug
```

### `app/Dockerfile.dev` (para hot reload)

```docker
FROM node:20-alpine

WORKDIR /app

# Instala nodemon globalmente
RUN npm install -g nodemon

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

# Nodemon reinicia ao detectar mudanças
CMD ["nodemon", "--watch", ".", "--ext", "js,json", "index.js"]
```

---

### 🚀 `docker-compose.prod.yml` (Produção)

```yaml
services:
  app:
    build:
      context: ./app
      dockerfile: Dockerfile  # Multi-stage otimizado
      args:
        - NODE_ENV=production
    image: registry.example.com/app:${APP_VERSION:-latest}  # Tag imutável
    pull_policy: if_not_present
    environment:
      - NODE_ENV=production
    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M
    # Restart policy para resiliência
    restart: unless-stopped
    # Logging com rotação
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    # Hardening de segurança
    read_only: true
    tmpfs:
      - /tmp
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

---

### 🔐 5. Gerenciando Variáveis com `.env`

### `.env.example` (commitar no Git)

```
# Copie para .env e preencha os valores sensíveis
DB_USER=app_user
DB_PASSWORD=
DB_NAME=app_db
APP_PORT=3000
NODE_ENV=production
APP_VERSION=1.0.0
```

### `.env` (NÃO commitar — adicione ao `.gitignore`)

```
DB_USER=app_user
DB_PASSWORD=S3cr3tPr0dP@ss!
DB_NAME=app_db
APP_PORT=3000
NODE_ENV=production
APP_VERSION=1.0.0-abc123
```

### Como usar:

```bash
# Desenvolvimento
docker compose --env-file .env.dev -f docker-compose.yml -f docker-compose.dev.yml up -d

# Produção
docker compose --env-file .env.prod -f docker-compose.yml -f docker-compose.prod.yml up -d

# Ou injete via shell (CI/CD)
export DB_PASSWORD=... && docker compose up -d
```

📌 **Nunca commitar `.env` com segredos.** Use:

- `.gitignore`: `.env`, `.local`, `secrets/`
- CI/CD: secrets do GitHub Actions, GitLab CI, etc.
- Produção: Docker Secrets, HashiCorp Vault, AWS Secrets Manager.

---

### 🧪 6. Validação & Testes

```bash
# 1. Build e suba em dev (com hot reload)
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build

# 2. Teste a API
curl http://localhost:3000

# 3. Altere um arquivo no VS Code → veja o navegador atualizar sem rebuild

# 4. Valide o cache
echo "// test" >> app/index.js
docker compose -f docker-compose.yml -f docker-compose.dev.yml build app
# → Deve ver "CACHED" nas camadas de npm install

# 5. Compare tamanhos de imagem
docker images | grep app
# → dev: ~300MB (com ferramentas) | prod: ~50MB (multi-stage)
```

---

### 💡 Cheat Sheet: Boas Práticas Consolidadas

| Prática | Por que fazer | Impacto |
| --- | --- | --- |
| `COPY package*.json` antes de `COPY . .` | Cache de dependências não invalida com mudanças de código | Build 10x mais rápido em dev |
| `.dockerignore` | Evita enviar lixo para o daemon, reduz contexto | Build mais rápido + segurança |
| `RUN` combinado com `&&` | Reduz número de camadas | Imagem menor, mais limpa |
| `--only=production` no `npm install` | Não instala devDependencies | Imagem menor, menos vulnerabilidades |
| `USER node` (não-root) | Segurança: limita danos se o container for comprometido | Hardening básico |
| Multi-stage builds | Separa build de runtime | Imagem final 80% menor |
| `:cached` em bind mounts (macOS/Windows) | Otimiza performance de leitura do host | Hot reload mais responsivo |
| `healthcheck` + `condition: service_healthy` | Garante ordem real de prontidão | Evita erros de conexão no startup |
| `read_only: true` + `cap_drop: ALL` | Princípio do menor privilégio | Reduz superfície de ataque |

---

### 🧭 Como Rodar na Prática

| Cenário | Comando |
| --- | --- |
| **Dev local** | `docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d` |
| **Prod local (teste)** | `docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d` |
| **CI/CD Build** | `docker compose -f docker-compose.yml -f docker-compose.prod.yml build --push` |
| **Debug específico** | `docker compose --profile debug -f docker-compose.yml -f docker-compose.dev.yml up -d` |

### Opcional: Alias no `Makefile`

```makefile
.PHONY: up-dev up-prod down logs build-prod

up-dev:
	docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build

up-prod:
	docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build

down:
	docker compose down

logs:
	docker compose logs -f app

build-prod:
	docker compose -f docker-compose.yml -f docker-compose.prod.yml build --no-cache
```

---