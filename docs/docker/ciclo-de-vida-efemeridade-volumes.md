# Ciclo de Vida, Efemeridade & Volumes

> **Objetivo:** Entender por que containers param, por que dados somem, como recuperar containers existentes e como desacoplar dados do ciclo de vida usando volumes.
> 

### 🔄 1. Ciclo de Vida & O Papel do PID 1 (10 min)

Um container **não é uma máquina ligada**. Ele é um processo isolado que vive **enquanto o processo principal (PID 1) estiver rodando**. Quando o PID 1 termina, o container para.

```bash
# 1. Sobe um container em background
docker run -d --name tmp alpine sleep 300
# → Retorna ID imediatamente. O terminal fica livre.

# 2. Verifique se está vivo
docker ps | grep tmp
# → STATUS: Up 5 seconds
```

🔍 **Por que `sleep 300`?**

O Alpine é minimalista. Se você rodar `docker run -d alpine echo "oi"`, o `echo` executa e termina em milissegundos. Como o PID 1 acabou, o container para. `sleep 300` é um "processo de vida" que mantém o container ativo por 5 minutos. Em produção, o PID 1 é seu app (`node`, `nginx`, `java`, etc.).

```bash
# 3. Pare o container graciosamente
docker stop tmp
# → Envia SIGTERM ao PID 1. O container finaliza.

# 4. Tente listá-lo
docker ps | grep tmp
# → (nada) Ele sumiu!

# 5. Liste TODOS os containers (incluindo parados)
docker ps -a | grep tmp
# → STATUS: Exited (0) 10 seconds ago
```

🧠 **Conceito-chave:**

`docker ps` = só rodando. `docker ps -a` = histórico completo.

Um container `Exited` **ainda existe no disco** (como um serviço parado no Windows). Ele não foi deletado.

---

### ⚠️ 2. Colisão de Nomes & `run` vs `start` (5 min)

Muitos iniciantes travam aqui. Tente rodar o mesmo nome novamente:

```bash
docker run -d --name tmp alpine sleep 300
# → docker: Error response from daemon: Conflict. The container name "/tmp" is already in use...
```

🔍 **Por que?**

O Docker garante unicidade de nomes no host. Mesmo parado, o nome `tmp` está reservado.

✅ **Duas soluções:**

| Ação | Comando | Quando usar |
| --- | --- | --- |
| **Recriar do zero** | `docker rm tmp` → depois `docker run ...` | Quando mudou `Dockerfile`, variáveis ou quer limpar estado |
| **Retomar o existente** | `docker start tmp` → `docker ps` | Quando só quer ligar de novo sem recriar filesystem |

```bash
# Retomando o container parado
docker start tmp
docker ps | grep tmp  # → STATUS: Up X seconds
```

---

### 🌊 3. Efemeridade do Filesystem (10 min)

Agora vamos gravar dados **dentro** do container e ver o que acontece.

```bash
# 1. Crie o diretório (Alpine não tem /data por padrão)
docker exec tmp sh -c "mkdir -p /data && echo 'vital' > /data/log.txt"

# 2. Valide
docker exec tmp cat //data/log.txt
# → vital ✅
```

!!! tip
    ### ⚠️ Nota sobre uso de caminhos no Windows (Git Bash + Docker)

    Ao executar comandos do Docker no Windows usando o Git Bash, caminhos no formato Unix (como `/data/log.txt`) podem ser automaticamente convertidos para caminhos do Windows (ex: `C:/Program Files/Git/data/log.txt`).

    Essa conversão é feita pelo ambiente MSYS (incluído no Git for Windows) e pode causar erros quando o comando é executado dentro de containers, onde o caminho original já está correto.

    ### ✔️ Como evitar o problema

    Use uma das alternativas abaixo:

    - Desativar a conversão de path no comando:

    ```
    MSYS_NO_PATHCONV=1 docker exec <container>cat /data/log.txt
    ```

    - Ou usar caminho com escape:

    ```
    docker exec <container>cat //data/log.txt
    ```

    ### 📌 Observação

    Esse ajuste é necessário apenas no Windows com Git Bash. Em ambientes Linux ou WSL, o comando funciona normalmente sem modificações.

🔍 **Onde isso está fisicamente?**

Na **camada gravável** (`writable layer`) do container. Cada container ganha uma camada de escrita sobre as camadas `read-only` da imagem.

```bash
# 3. Simule "deploy" ou "falha": pare e REMOVA o container
docker stop tmp && docker rm tmp

# 4. Crie um NOVO container com a MESMA imagem
docker run -d --name tmp alpine sleep 300

# 5. Tente ler o arquivo
docker exec tmp cat /data/log.txt
# → cat: can't open '/data/log.txt': No such file or directory
```

🧠 **Conceito-chave:**

Containers são **stateless por design**. A camada gravável é atrelada à instância do container. Quando você `rm` o container, essa camada é destruída. O próximo `run` cria um filesystem **limpo e idêntico** à imagem original.

✅ **Bom para:** APIs, workers, frontend (tudo que pode ser recriado).

❌ **Catastrófico para:** Bancos de dados, uploads, sessões, logs de auditoria.

---

### 📦 4. Volumes: Desacoplando Dados do Ciclo de Vida (15 min)

Para persistir dados, precisamos **bypassar a camada gravável do container** e escrever diretamente em um armazenamento externo ao ciclo de vida.

### 🔹 A. Bind Mount (Host ↔ Container)

Espelha um caminho real do seu SO dentro do container. Ideal para **desenvolvimento**.

```bash
# 1. Crie pasta no host
mkdir -p docker-demo/vol-host

# 2. Sobe container mapeando a pasta
docker run -d --name bind-test -v $(pwd)/docker-demo/vol-host:/dados alpine sleep 300

# 3. Escreva DENTRO do container
docker exec bind-test sh -c "echo 'persiste no host' > /dados/arquivo.txt"

# 4. Valide NO SEU HOST (abre outro terminal ou use IDE)
cat docker-demo/vol-host/arquivo.txt
# → persiste no host ✅
```

🔍 **Por que funciona?**

O bind mount monta o filesystem do host diretamente no container. O container só aponta para ele. Se o container morrer, o arquivo continua no seu disco.

```bash
# 5. Remova e recrie
docker rm -f bind-test
docker run -d --name bind-test -v $(pwd)/docker-demo/vol-host:/dados alpine sleep 300

# 6. O dado sobreviveu
docker exec bind-test cat /dados/arquivo.txt
# → persiste no host ✅
```

⚠️ **Cuidado:** Bind mounts herdam permissões do host. No Linux, se o container roda como `root` e você como `user 1000`, pode dar `Permission denied`. Use `:z`/`:Z` ou ajuste `uid/gid` em produção.

### 🔹 B. Named Volume (Gerenciado pelo Docker)

O Docker cria e gerencia o armazenamento em `/var/lib/docker/volumes/`. Ideal para **bancos, caches, produção**.

```bash
# 1. Crie volume gerenciado
docker volume create pg_data

# 2. Subir Postgres usando o volume
docker run -d --name db-test -v pg_data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=secret postgres:16-alpine

# 3. Aguarde inicialização (~5s)
docker logs db-test 2>&1 | grep "ready"
```

🔍 **Vantagens sobre bind mount:**

- Docker gerencia `ownership`, `filesystem` (ext4/xfs) e performance otimizada.
- Funciona igualmente em Linux, Windows e macOS (sem problemas de path/permissions).
- Fácil backup: `docker run --rm -v pg_data:/data -v $(pwd):/backup alpine tar czf /backup/pg.tar.gz -C /data .`

```bash
# 4. Inspecione e gerencie
docker volume ls
docker volume inspect pg_data
# → Mountpoint: /var/lib/docker/volumes/pg_data/_data

# 5. Limpeza
docker rm -f db-test
docker volume rm pg_data  # ⚠️ Apaga os dados permanentemente
```

📌 **Regra Prática:**

| Tipo | Sintaxe | Uso Ideal |
| --- | --- | --- |
| `Bind Mount` | `-v ./host:/container` | Código, configs, hot-reload, dev local |
| `Named Volume` | `-v nome:/container` | Postgres, Redis, Mongo, uploads, prod |

---

### 🧪 Experimentos Guiados (para a turma fazer junto)

1. **Teste `start` vs `run`:**
    
    ```bash
    docker stop tmp
    docker run -d --name tmp alpine sleep 300  # → falha (nome em uso)
    docker start tmp                           # → funciona
    docker ps | grep tmp                       # → Up
    ```
    
2. **Veja o filesystem mudando:**
    
    ```bash
    docker exec tmp ls /
    docker exec tmp mkdir -p /app && echo "oi" > /app/test.txt
    docker exec tmp ls /app
    docker rm -f tmp
    docker run -d --name tmp alpine sleep 300
    docker exec tmp ls /app  # → (vazio, filesystem resetado)
    ```
    
3. **Compare permissões:**
    
    ```bash
    ls -la ~/docker-demo/vol-host/
    docker exec bind-test ls -la /dados/
    # → Note como uid/gid podem divergir entre host e container
    ```
    

---

### 📋 Checklist de Comandos do Bloco

```bash
docker ps / ps -a          # Rodando vs Todos
docker stop/start <nome>   # Para/Retoma (não deleta)
docker rm <nome>           # Deleta container (libera nome)
docker run -d --name <n>   # Cria novo (exige nome livre)
docker exec <n> sh         # Terminal cirúrgico
docker volume create/ls/rm # Gerencia volumes nomeados
```

---

### 🧹 Limpeza Antes de Continuar

```bash
docker rm -f $(docker ps -aq)
docker volume rm pg_data 2>/dev/null || true
```