# 🐳 DOCKER & DOCKER COMPOSE – CHEAT SHEET

*(Encontro 1 | Imprima ou mantenha aberto na segunda tela)*

---

## 🔑 CONCEITOS FUNDAMENTAIS

| Conceito | Explicação Rápida |
| --- | --- |
| **VM vs Container** | VM = SO completo + hardware virtual (pesado). Container = processo isolado usando kernel do host (leve, boot em ms). |
| **Imagem ≠ Container** | Imagem = camadas `read-only` (receita). Container = instância em execução + camada gravável no topo (bolo). |
| **Efemeridade** | Containers são **stateless**. Tudo escrito dentro deles **some** ao parar/remover. Use volumes para persistir. |

---

## 📦 VOLUMES: QUANDO USAR?

| Tipo | Sintaxe CLI | Uso Ideal | Observação |
| --- | --- | --- | --- |
| **Bind Mount** | `-v ./host:/container` | Código, configs, hot-reload (Dev) | Espelha host ↔ container. Cuidado com permissões no Linux. |
| **Named Volume** | `-v nome:/container` | Bancos, uploads, caches, filas | Docker gerencia `uid`, `fs` e backup. Melhor performance. |

📌 **Compose:** `volumes: ["./html:/usr/share/nginx/html:ro"]` ou `volumes: ["pg_data:/var/lib/postgresql/data"]`

---

## 🔁 CLI → DOCKER COMPOSE (Tradução Direta)

| Flag `docker run` | Campo `docker-compose.yml` |
| --- | --- |
| `-p 8080:80` | `ports: ["8080:80"]` |
| `-e VAR=val` | `environment: [VAR=val]` |
| `-v nome:/path` | `volumes: ["nome:/path"]` |
| `--network net` | `networks: ["net"]` |
| `--name svc` | `services: svc:` |
| `--depends-on svc` | `depends_on: ["svc"]` |

⚠️ **REGRAS DE OURO DO COMPOSE**

1. Cria rede interna automática. Serviços se resolvem por **nome** (ex: `DB_HOST=db`).
2. `localhost` ou `127.0.0.1` **nunca** funciona entre containers.
3. `depends_on` só espera o container **iniciar**, não o serviço estar pronto.

---

## 🛠️ COMANDOS ESSENCIAIS

| Categoria | Comando | O que faz |
| --- | --- | --- |
| **Ciclo de Vida** | `docker run -d --name nome -p host:container imagem` | Sobe container em background |
|  | `docker stop/start/rm nome` | Para / inicia / remove |
|  | `docker ps [-a]` | Lista containers (rodando / todos) |
| **Imagens** | `docker build -t nome:tag .` | Gera imagem a partir do `Dockerfile` |
|  | `docker images` / `docker rmi imagem` | Lista / remove imagens |
| **Debug** | `docker logs [-f] nome` | Vê saída padrão (use `-f` para seguir) |
|  | `docker exec -it nome sh` | Entra no container (use `exit` para sair) |
|  | `docker inspect nome` | JSON completo (IPs, mounts, config) |
|  | `docker stats` | CPU/MEM/NET em tempo real |
| **Compose** | `docker compose up -d [--build]` | Sobe tudo em background (rebuild se mudar `Dockerfile`) |
|  | `docker compose down [-v]` | Para containers (`-v` remove volumes nomeados) |
|  | `docker compose logs -f [servico]` | Logs em tempo real (filtre por serviço) |
|  | `docker compose ps / exec / config` | Estado, terminal interativo, valida YAML |
| **Limpeza** | `docker system df` | Espaço usado por Docker |
|  | `docker system prune -f` | Remove containers/imagens/networks órfãos (seguro) |

---

## 🚨 ERROS COMUNS & COMO RESOLVER

| Sintoma | Causa Provável | Solução |
| --- | --- | --- |
| `ECONNREFUSED 127.0.0.1:5432` | `DB_HOST=localhost` no container | Troque pelo nome do serviço: `DB_HOST=db` |
| `port is already allocated` | Porta host já em uso | `docker ps` para identificar ou mude `8081:80` |
| `Permission denied` em bind mount | Container roda como `root` | `docker run --user $(id -u):$(id -g)` ou use named volume |
| Dados sumiram após `down` | Usou `docker compose down -v` | Use só `down` no dia a dia. `-v` reseta o DB. |
| Hot reload não funciona | Faltou bind mount ou watcher | Adicione `./src:/app` e `CHOKIDAR_USEPOLLING=true` |

---

## ✅ BOAS PRÁTICAS RÁPIDAS

- 📄 Sempre use `.dockerignore` (`node_modules`, `.git`, `.log`, `.env`)
- ⏳ Containers efêmeros: adicione `--rm` se não precisar manter o histórico
- 🔒 Não rode como `root` em produção (`USER node` no `Dockerfile`)
- 📦 Compose v2+: **não precisa** de `version: "3.9"` (é legado)
- 🩺 Para produção, substitua `depends_on` por `healthcheck` + `condition: service_healthy`
- 🪟 Windows/WSL: `$(pwd)` não funciona no CMD. Use `${PWD}` (PowerShell) ou rode dentro do WSL.

---

## 🧭 FLUXO DE DEBUG (SEMPRE NA ORDEM)

1. `docker compose logs -f <serviço>` → O que o processo está reclamando?
2. `docker compose ps` → Container está `Up` ou `Exited`?
3. `docker compose exec <serviço> sh` → Teste manualmente dentro do container.
4. `docker compose config` → YAML está sendo interpretado corretamente?
5. `docker system prune -f` → Espaço cheio travando o daemon?

📌 **Dica de Ouro:** `logs` primeiro. `inspect` depois. `exec` por último.

---

🖨️ *Pronto para imprimir (1 página frente e verso).*

📖 Docs oficiais: `docs.docker.com` | 💡 Canal de dúvidas: envie `1) erro exato, 2) logs, 3) trecho do YAML`