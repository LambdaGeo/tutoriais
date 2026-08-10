# Tradução CLI → Docker Compose (Construção Passo a Passo)

> **Filosofia:** Docker Compose não é magia. É **CLI declarativa**. Cada flag vira um campo YAML. Vamos construir o arquivo juntos, serviço por serviço.
> 

### 🔹 1. O Problema da CLI (Motivação)

Mostre na tela:

```bash
docker run -d --name web -p 8080:80 -v $(pwd)/html:/usr/share/nginx/html:ro nginx:alpine
docker run -d --name cache -v cache_data:/data redis:alpine
docker run -d --name app -p 3000:3000 -e DB_HOST=cache --network app-net --depends-on cache node:20-alpine
```

🗣️ **Pergunte à turma:**

- *"Como compartilhar isso com outro dev?"*
- *"E se eu precisar subir isso em CI/CD?"*
- *"Como garantir que `app` só inicie depois que `cache` estiver pronto?"*
- *"E se eu esquecer uma flag na 3ª linha?"*

✅ **Resposta:** Precisamos de um **manifesto declarativo**, versionável no Git, que o Docker Engine execute de forma idempotente. Esse manifesto é o `docker-compose.yml`.

---

### 🔹 2. Mapeamento Direto (CLI ↔ YAML)

Antes de escrever, mostre a tradução exata. Cole isso no slide/README:

| Flag `docker run` | Campo `docker-compose.yml` | O que faz |
| --- | --- | --- |
| `--name web` | `services: web:` | Define o identificador do serviço |
| `nginx:alpine` | `image: nginx:alpine` | Imagem base |
| `-p 8080:80` | `ports: ["8080:80"]` | Mapeamento Host → Container |
| `-e DB_HOST=cache` | `environment: DB_HOST: cache` | Injeção de variáveis |
| `-v ./html:/path` | `volumes: ["./html:/path"]` | Bind Mount |
| `-v cache_data:/path` | `volumes: ["cache_data:/path"]` | Named Volume |
| `--network app-net` | `networks: ["app-net"]` | Anexa à rede |
| `--depends-on cache` | `depends_on: [cache]` | Ordem de criação |

📌 **Regra de Ouro do Compose:**

- Não use `version: "3.9"` (legado desde Compose v2).
- O Compose **cria automaticamente** uma rede chamada `<projeto>_default`. Você só declara `networks:` se quiser isolamento extra ou nomes customizados.
- Todos os serviços na mesma rede resolvem DNS por **nome do serviço**.

---

## 📁 Passo 0: Preparação do Projeto

Crie a estrutura inicial na sua máquina:

```bash
mkdir compose-step-by-step && cd compose-step-by-step
mkdir html
echo "<h1>Compose Funcionando!</h1>" > html/index.html
touch docker-compose.yml
```

!!! tip
    Pode criar esses arquivos e diretorio usando o vscode

📝 **Abra `docker-compose.yml` no editor e comece com o esqueleto vazio:**

```yaml
services:
```

💡 **Conceito:** `services:` é o nível raiz. Cada chave abaixo dele será um container gerenciado pelo Compose.

---

## 🟢 Passo 1: Serviço Web (Nginx)

### 1️⃣ Adicione o serviço `web`

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
```

### 🔍 Explicação de cada chave:

| Chave | O que faz |
| --- | --- |
| `image: nginx:alpine` | Baixa e usa a imagem oficial. Se não existir localmente, o Docker puxa do Docker Hub. |
| `ports: ["8080:80"]` | Mapeia a porta `8080` da sua máquina para a porta `80` dentro do container. Formato: `host:container`. |
| `volumes: ["./html:/usr/share/nginx/html:ro"]` | Bind mount. Espelha a pasta `html` do host no container. `:ro` = read-only (segurança + performance). |

### 2️⃣ Valide a configuração

```bash
docker compose config
```

📤 **Output esperado (trecho relevante):**

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - published: 8080
        target: 80
    volumes:
      - type: bind
        source: /caminho/absoluto/para/compose-step-by-step/html
        target: /usr/share/nginx/html
        read_only: true
```

🧠 **O que o Compose fez:**

- Resolveu `./html` para o **caminho absoluto** do seu SO.
- Expandiu a sintaxe curta de `ports` e `volumes` para a forma longa e explícita.
- Adicionou automaticamente uma rede `default` (não aparece aqui, mas existe).

### 3️⃣ Levante e teste

```bash
docker compose up -d
docker compose ps
```

🌐 Abra `http://localhost:8080` → Deve aparecer `"Compose Funcionando!"`.

✅ **Checkpoint:** O primeiro serviço está no ar. O Compose criou a rede interna e o bind mount. Vamos adicionar o próximo.

---

## 🔴 Passo 2: Serviço Cache (Redis) + Declaração Obrigatória de Volume

### 1️⃣ Adicione o serviço `cache` E declare o volume raiz

O Compose v2 valida rigidamente a estrutura. Para evitar o erro `undefined volume`, você deve declarar o volume no final do arquivo **antes** ou **junto** com o uso.

Atualize seu `docker-compose.yml` para:

```yaml
services:
  web:
    image: nginx:alpine
    ports: ["8080:80"]
    volumes: ["./html:/usr/share/nginx/html:ro"]

  cache:
    image: redis:alpine
    environment:
      REDIS_MAXMEMORY: 256mb
    volumes:
      - cache_data:/data

volumes:
  cache_data:  # ← OBRIGATÓRIO para named volumes
```

### 🔍 Explicação de cada chave:

| Chave | O que faz |
| --- | --- |
| `image: redis:alpine` | Imagem oficial leve. |
| `environment:` | Injeta variáveis no container. ⚠️ Redis ignora `REDIS_MAXMEMORY` nativamente. É um exemplo didático de injeção. Em prod, use `redis.conf`. |
| `volumes: ["cache_data:/data"]` | **Named Volume**. O Docker cria e gerencia o armazenamento. O container acessa via `/data`. |
| `volumes:` (bloco raiz) | **Declaração raiz obrigatória**. Informa ao Compose que `cache_data` é um recurso gerenciado, não um caminho do host. |

📌 **Regra de Ouro do Compose:**

> Se tem dois-pontos no volume (`nome:/path`), declare no bloco raiz `volumes:`. Se não tem (`./pasta:/path`), é bind mount e não precisa declarar.

### 2️⃣ Valide a configuração

```bash
docker compose config
```

📤 **Output esperado (trecho relevante):**

```yaml
services:
  cache:
    image: redis:alpine
    volumes:
      - type: volume
        source: cache_data
        target: /data
volumes:
  cache_data:
    name: tutorialdocker_cache_data
    driver: local
```

🧠 **O que o Compose fez:**

- Validou que `cache_data` existe na seção raiz.
- Resolveu o caminho interno para `/data` (absoluto, como exige o daemon).
- Prefixou o nome com o projeto (`tutorialdocker_`) para evitar colisão global.

### 3️⃣ Levante incrementalmente e teste

```bash
docker compose up -d
```

📤 **Output esperado:**

```
[+] Running 3/3
 ✔ Network tutorialdocker_default  Created
 ✔ Volume  tutorialdocker_cache_data  Created
 ✔ Container tutorialdocker-cache-1  Started
```

🔍 **Note:** O Compose **não recriou** o `web`. Só criou o volume, anexou à rede e subiu o `cache`.

✅ **Teste rápido:**

```bash
docker compose exec cache redis-cli ping
# → PONG
docker compose exec cache env | grep REDIS
# → REDIS_MAXMEMORY=256mb
```

---

### 💡 Por que o Compose exige isso? (Para explicar na aula)

1. **Segurança de parsing:** Sem a declaração raiz, o Compose não saberia se `cache_data` é um volume gerenciado ou se você esqueceu o `:` de um bind mount.
2. **Extensibilidade:** No futuro, você pode adicionar `driver: nfs`, `labels:`, ou `external: true` direto nessa declaração.
3. **Previsibilidade:** Garante que todos os recursos (redes, volumes, configs) são explicitamente declarados antes do deploy.

---

✅ **Checkpoint:** Dois serviços rodando na mesma rede automática. Dados do Redis serão persistidos no volume nomeado.

---

## 🟡 Passo 3: Serviço App (Node)

### 1️⃣ Prepare a estrutura

Na raiz do projeto (`tutorialdocker/`), crie a pasta:

```bash
mkdir app
```

Agora, **abra o VS Code** e crie os 3 arquivos abaixo dentro da pasta `app/`.

---

### 📄 `app/package.json`

```json
{
  "name": "app",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  }
}
```

🔍 **O que faz:** Define o ponto de entrada (`index.js`) e o comando que será executado no `Dockerfile`.

---

### 📄 `app/index.js`

```jsx
const http = require('http');

const server = http.createServer((req, res) => {
res.writeHead(200, { 'Content-Type': 'application/json' });
res.end(JSON.stringify({
status: 'online',
db_host: process.env.DB_HOST || 'não configurado',
timestamp: new Date().toISOString()
}));
});

// 0.0.0.0 é OBRIGATÓRIO para o Docker mapear a porta corretamente
server.listen(3000, '0.0.0.0', () => {
console.log('✅ App ouvindo em http://0.0.0.0:3000');
console.log('🔗 DB_HOST:', process.env.DB_HOST);
});

// Mantém o processo vivo para o container não parar
setInterval(() => {}, 1000);
```

🔍 **Por que `'0.0.0.0'`?**Se o Node ouvir apenas `127.0.0.1` (padrão de alguns frameworks), a requisição fica presa na interface de *loopback* interna do container. `0.0.0.0` instrui o processo a escutar em **todas as interfaces de rede**, permitindo que o Docker Engine redirecione a porta do host.

📦 **Nota:** Este código usa apenas o módulo nativo `http`. Não precisa instalar `express`. O `package.json` anterior já funciona perfeitamente.

---

### 📄 `app/Dockerfile`

```docker
FROM node:20-alpine
WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
EXPOSE 3000

CMD ["npm", "start"]
```

🔍 **O que faz:**

- `FROM node:20-alpine`: Imagem leve (~50MB).
- `COPY package*.json ./` + `RUN npm install`: Aproveita cache do Docker. Só reinstala se `package.json` mudar.
- `CMD ["npm", "start"]`: Processo principal (`PID 1`) que mantém o container vivo.

---

### 🔄 2. Atualize o `docker-compose.yml`

Adicione o bloco `app` **abaixo** do `cache`. O arquivo completo deve ficar assim:

```yaml
services:
  web:
    image: nginx:alpine
    ports: ["8080:80"]
    volumes: ["./html:/usr/share/nginx/html:ro"]

  cache:
    image: redis:alpine
    environment:
      REDIS_MAXMEMORY: 256mb
    volumes:
      - cache_data:/data

  app:
    build: ./app
    ports: ["3000:3000"]
    environment:
      DB_HOST: cache
      APP_ENV: development
    depends_on:
      - cache

volumes:
  cache_data:
```

🔍 **Explicação das novas chaves:**

| Chave | O que faz |
| --- | --- |
| `build: ./app` | O Compose executa `docker build ./app` automaticamente antes de subir. |
| `environment: DB_HOST: cache` | Injeta a variável. `cache` é o **nome do serviço**, não um IP. O DNS interno do Compose resolve. |
| `depends_on: [cache]` | Garante que o container `cache` seja criado antes do `app`. ⚠️ Só garante *criação*, não *prontidão*. |

---

### 🛠️ 3. Valide e Levante

```bash
# 1. Verifique se o YAML está sintaticamente correto e resolveu os caminhos
docker compose config

# 2. Suba tudo (o Compose vai fazer build do app automaticamente)
docker compose up -d --build
```

📤 **Output esperado:**

```
[+] Running 4/4
 ✔ Network tutorialdocker_default        Created
 ✔ Volume  tutorialdocker_cache_data     Created
 ✔ Container tutorialdocker-cache-1      Started
 ✔ Container tutorialdocker-app-1        Started
 ✔ Container tutorialdocker-web-1        Started
```

### ✅ 4. Teste DNS Interno & Logs

```bash
# Veja o que o app imprimiu ao iniciar
docker compose logs app
```

📤 **Output esperado:**

```
app-1  | ✅ App ouvindo em http://0.0.0.0:3000
app-1  | 🔗 DB_HOST: cache
```

```bash
# Prove que o DNS interno está funcionando
docker compose exec app sh -c "nslookup cache"
# → Name: cache
# → Address 1: 172.18.0.3 (IP dinâmico, pode variar)
```

🧠 **Conceito fixado:**

O Compose criou uma rede `default`, anexou os 3 serviços nela, e o DNS interno resolveu `cache` → IP real do container Redis. O `app` não precisa saber IPs. Só precisa do **nome do serviço**.

---

### 🛠️ Como validar que o `app` está rodando

```bash
# 1. Rebuild e suba apenas o app (incremental)
docker compose up -d --build app

# 2. Teste via terminal
curl http://localhost:3000
# → {"status":"online","db_host":"cache","timestamp":"2026-04-30T..."}

# 3. Ou abra no navegador: http://localhost:3000
```

📤 **Output esperado nos logs:**

```bash
docker compose logs app
# → ✅ App ouvindo em http://0.0.0.0:3000
# → 🔗 DB_HOST: cache
```

📌 **Conceito-chave para fixar:**

| Item | O que significa |
| --- | --- |
| `EXPOSE 3000` no Dockerfile | Apenas **documentação**. Não abre porta. |
| `ports: ["3000:3000"]` no Compose | Cria a regra de NAT no Docker Engine. |
| `server.listen(3000, '0.0.0.0')` no código | **Ativa o listener** no container. Sem isso, o mapeamento de porta não tem para onde redirecionar. |

---

### ✅ Checklist de Validação Final do Stack

```bash
docker compose ps
# → web, cache, app devem estar "Up"

curl http://localhost:8080   # ✅ Nginx (web)
curl http://localhost:3000   # ✅ Node (app)
docker compose exec cache redis-cli ping  # ✅ Redis (cache)
```

---