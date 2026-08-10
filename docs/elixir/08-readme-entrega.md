# 📄 Fase 8: README e Entrega

**Objetivo:** transformar o `README.md` padrão do Phoenix em uma documentação de verdade e fazer o checklist final do repositório.

**Por que fazemos isso?**
O README é a porta de entrada de qualquer repositório — o primeiro (às vezes o único) arquivo que um colega, recrutador ou avaliador vai ler. O critério de qualidade é simples: _alguém conseguiria clonar e rodar seu projeto lendo apenas o README?_

### 📝 Passo 8.1: Atualizar o `README.md`

O `phx.new` já gerou um `README.md` genérico na raiz do projeto. Vamos substituí-lo por um personalizado. Adapte o modelo (nome, links, o que quiser):

````markdown
# Todo List — Elixir + Phoenix LiveView

**Aluno(a):** Seu Nome Completo Aqui

**Tutorial original:** [Como Criar um App "Todo List" com Elixir e LiveView
do Zero](https://github.com/LambdaGeo/tutoriais)

## Descrição

Aplicação de lista de tarefas (Todo List) construída de forma incremental
para estudar a arquitetura **full-stack funcional reativa** do
[Phoenix LiveView](https://hexdocs.pm/phoenix_live_view):

- **Framework:** Phoenix 1.8 + LiveView 1.1 — o estado vive no servidor e a
  interface atualiza em tempo real via WebSocket, sem JavaScript manual;
- **Persistência:** [Ecto](https://hexdocs.pm/ecto) + SQLite (schema,
  migrations e validação com changesets);
- **Visual:** Tailwind CSS v4 + daisyUI.

Funcionalidades: criar tarefas (com validação), listar, marcar como
concluída (toggle) e excluir — tudo com atualização instantânea.

## Pré-requisitos

| Ferramenta | Versão    |
| ---------- | --------- |
| Erlang/OTP | 26+       |
| Elixir     | 1.17+     |
| Node.js    | 18+ (LTS) |

## Como Rodar

```bash
git clone https://github.com/SEU-USUARIO/SEU-REPO.git
cd SEU-REPO

# Instala dependências e prepara os assets
mix setup

# Cria o banco SQLite e aplica as migrations
mix ecto.create
mix ecto.migrate

# Sobe o servidor
mix phx.server
```

Depois, abra [http://localhost:4000](http://localhost:4000) no navegador.

## Estrutura Principal

| Arquivo                                      | Papel                               |
| -------------------------------------------- | ----------------------------------- |
| `lib/elixir_todo_list_web/live/todo_live.ex` | O LiveView (estado, eventos e HTML) |
| `lib/elixir_todo_list/task.ex`               | O schema `Task` e seu changeset     |
| `lib/elixir_todo_list/repo.ex`               | A conexão com o banco (Ecto.Repo)   |
| `priv/repo/migrations/`                      | As migrations do banco              |
````

### 🧪 Passo 8.2: O Teste do "Colega"

Faça de conta que você é outra pessoa: clone o seu repositório em uma **pasta nova** (ex: `/tmp/teste-todo`) e siga o seu próprio README, literalmente. A aplicação subiu? O CRUD funciona? Os dados persistem após reiniciar o servidor?

Esse teste revela os dois vacilos clássicos:

1. **Banco commitado**: se `git status`/a página do GitHub mostrar `elixir_todo_list.db` (ou `.db-shm`/`.db-wal`), o `.gitignore` da Fase 0 não foi aplicado antes do commit. Corrija com:

   ```bash
   git rm --cached elixir_todo_list.db elixir_todo_list.db-shm elixir_todo_list.db-wal
   git commit -m "Remove arquivos de banco do versionamento"
   ```

2. **Passo de setup faltando** no README (ex: esquecer o `mix ecto.migrate` — sem ele, a aplicação sobe mas quebra ao carregar as tarefas).

### 💾 Passo 8.3: Commit Final

```bash
git add README.md
git commit -m "Fase 8: Atualiza README com instruções de execução"
```

### ✅ Passo 8.4: Checklist Final

Confira o histórico:

```bash
git log --oneline
```

Esperado (de cima para baixo):

```
Fase 8: Atualiza README com instruções de execução
Fase 7: Ajusta o tema e personaliza o visual (Tailwind/daisyUI)
Fase 6: Implementa conclusão de tarefas (toggle_complete)
Fase 5: Implementa exclusão de tarefas (delete)
Fase 4: Refatora TodoLive para usar Ecto, Repo e to_form()
Fase 3: Persistência - Configura Ecto, Repo, Migrations e Task Schema
Fase 2: Lógica em Memória - Implementa adição de tarefas
Fase 1: Prova de Vida - Substitui a rota raiz por TodoLive
Fase 0: Gera o esqueleto do Phoenix com LiveView (sem Ecto)
Fase 0: Inicializa o repositório e .gitignore
```

**Checklist:**

- [ ] O repositório no GitHub é público e o `git push` foi feito?
- [ ] Nenhum arquivo `*.db`, `*.db-shm` ou `*.db-wal` aparece no GitHub?
- [ ] As pastas `_build/` e `deps/` também não aparecem?
- [ ] O README tem nome, link do tutorial, descrição e instruções que funcionam?
- [ ] O CRUD completo funciona e os dados sobrevivem ao restart do servidor?

---

## 🚀 Tutorial Concluído!

**Parabéns!** 🥳 Você construiu uma aplicação web **full-stack funcional e reativa** — e, se fez também o tutorial de Clojure, agora viu **o mesmo problema resolvido por duas filosofias**:

|                                 | Clojure                  | Elixir                         |
| ------------------------------- | ------------------------ | ------------------------------ |
| Onde vive o estado reativo      | No navegador (`r/atom`)  | No servidor (socket/`assigns`) |
| Como a UI conversa com os dados | API REST + JSON + CORS   | WebSocket interno (`phx-*`)    |
| Onde a validação acontece       | No handler, "à mão"      | No changeset, declarativa      |
| Quantos servidores em dev       | Dois (API + shadow-cljs) | Um (`mix phx.server`)          |

**Quer ir além?** Algumas extensões (opcionais):

- Um filtro "Todas / Ativas / Concluídas" (um `assign` + cláusulas de `handle_event`);
- Edição do título de uma tarefa (outro pequeno `<.form>` por item);
- Um contador "X de Y concluídas" no topo;
- Trocar o SQLite por PostgreSQL (`ecto_sqlite3` → `postgrex`, e o adapter no Repo);
- Explorar o **Phoenix PubSub** para que duas abas do navegador vejam as mudanças uma da outra em tempo real — a "mágica" concorrente da BEAM em ação.
