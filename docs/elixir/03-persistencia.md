# 🧱 Fase 3: Persistência — A Camada de Dados (Ecto, Repo, Migration e Schema)

Na fase anterior, nossa aplicação mantinha as tarefas apenas **em memória**. Agora daremos um passo importante: **armazenar os dados de forma permanente** usando o **Ecto**, a biblioteca de persistência do Elixir.

Optamos por **não incluir o Ecto desde o início** (`--no-ecto`) para que você entendesse primeiro a lógica interna do LiveView. Agora veremos **como adicionar novas dependências** a um projeto existente e configurar o banco de dados real.

Nesta fase, montamos **apenas a camada de dados** (dependências, Repo, migration e schema) e a testamos isoladamente. O "transplante" — trocar o coração em memória do `TodoLive` pelo banco — fica para a **Fase 4**.

### ⚙️ Passo 3.1: Adicionar as Dependências do Ecto

Pare o servidor (`Ctrl+C` duas vezes) e abra o arquivo `mix.exs`, o equivalente ao `package.json` (JavaScript) ou `requirements.txt` (Python). É nele que declaramos as **dependências do projeto**, dentro da função `defp deps do`.

!!! warning
    **NÃO substitua a lista de dependências!** O `phx.new` já gerou uma lista grande (phoenix, phoenix_html, live_view, tailwind, esbuild, bandit e mais uma dúzia) — **todas são necessárias**. Nossa tarefa é apenas **acrescentar quatro linhas** a essa lista. Substituir a lista inteira por uma menor quebraria o projeto por completo.


**Ação:** localize `defp deps do` no `mix.exs` e **adicione** estas quatro linhas em qualquer ponto da lista (por exemplo, logo após a linha do `{:phoenix_live_view, ...}`), mantendo tudo o que já existe:

```elixir
  defp deps do
    [
      {:phoenix, "~> 1.8.1"},
      # ... (todas as dependências que o phx.new gerou — NÃO apague nada!) ...

      # --- ADICIONE ESTAS 4 LINHAS (suporte ao Ecto) ---
      {:ecto, "~> 3.11"},
      {:phoenix_ecto, "~> 4.4"},
      {:ecto_sql, "~> 3.10"},
      {:ecto_sqlite3, "~> 0.12"}, # SQLite: banco leve, baseado em arquivo
      # --------------------------------------------------

      # ... (o restante das dependências geradas) ...
    ]
  end
```

O que cada uma faz:

| Dependência    | Papel                                                                           |
| -------------- | ------------------------------------------------------------------------------- |
| `ecto`         | O núcleo: schemas, changesets e queries.                                        |
| `ecto_sql`     | A camada SQL do Ecto (migrations, adaptadores).                                 |
| `ecto_sqlite3` | O "driver" específico do SQLite.                                                |
| `phoenix_ecto` | A cola entre o Phoenix e o Ecto (ex: integração de changesets com formulários). |

Em seguida, baixe as dependências:

```bash
mix deps.get
```

### 🧩 Passo 3.2: Configurar o "Repo" — o Agente do Banco de Dados

O **Repo** (de _repository_) é o módulo que representa a **conexão com o banco**. É como o `settings.py` do Django ou o `database.yml` do Rails: ele sabe **onde está o banco e como acessá-lo**.

**Ação 1:** crie o arquivo `lib/elixir_todo_list/repo.ex`:

```elixir
defmodule ElixirTodoList.Repo do
  use Ecto.Repo,
    otp_app: :elixir_todo_list,
    adapter: Ecto.Adapters.SQLite3
end
```

**Ação 2:** agora precisamos **supervisionar** o Repo — garantir que ele seja iniciado (e reiniciado, se cair) junto com a aplicação. Abra `lib/elixir_todo_list/application.ex` e adicione o Repo **no início** da lista de processos supervisionados:

```elixir
    children = [
      ElixirTodoList.Repo, # 👈 ADICIONE ESTA LINHA (antes dos demais)
      ElixirTodoListWeb.Telemetry,
      {DNSCluster, query: Application.get_env(:elixir_todo_list, :dns_cluster_query) || :ignore},
      {Phoenix.PubSub, name: ElixirTodoList.PubSub},
      # ...
      ElixirTodoListWeb.Endpoint
    ]
```

!!! tip
    Essa lista `children` é a **árvore de supervisão** — um dos superpoderes do Elixir. Cada item é um processo que a aplicação inicia e vigia. Se o Repo travar, o supervisor o reinicia automaticamente. É o mesmo mecanismo que "ressuscitou" nosso LiveView na Fase 2!


**Ação 3:** por fim, configure o Repo no arquivo `config/config.exs`. Adicione (por exemplo, logo após a linha `import Config` no topo):

```elixir
config :elixir_todo_list, ElixirTodoList.Repo,
  database: "elixir_todo_list.db",
  priv: "priv/repo"

config :elixir_todo_list, ecto_repos: [ElixirTodoList.Repo]
```

Entendendo as opções:

- `database:` → o **arquivo** onde o SQLite salvará tudo. Com esse valor, ele será criado na **raiz do projeto** (`elixir_todo_list.db`).
- `priv:` → onde ficam os arquivos de apoio do Repo — em especial, as **migrations** (`priv/repo/migrations/`). _Atenção: isso não muda o local do banco!_
- `ecto_repos:` → informa às tarefas do Mix (`mix ecto.create`, `mix ecto.migrate`) quais Repos existem.

!!! tip
    Lembra do `.gitignore` da Fase 0? As regras `*.db`, `*.db-shm` e `*.db-wal` foram escritas exatamente para este momento: o banco (e seus arquivos auxiliares de escrita) vão aparecer na raiz do projeto e **não devem** ser versionados.


### 🏗️ Passo 3.3: Criar o Banco de Dados

O banco ainda não existe. Crie-o com:

```bash
mix ecto.create
```

**Resultado Esperado:**

```
The database for ElixirTodoList.Repo has been created
```

Olhe a raiz do projeto: o arquivo `elixir_todo_list.db` apareceu. (E rode `git status` para confirmar que ele **não** aparece para o Git — obrigado, `.gitignore`!)

### 🧬 Passo 3.4: Migrations — a "Planta Baixa" do Banco

Em praticamente todos os frameworks modernos (Django, Rails, Laravel), usamos **migrations** para versionar e aplicar mudanças no banco. Cada migration é um pequeno "passo evolutivo": criar uma tabela, adicionar uma coluna, etc.

**Ação 1:** gere a migration da nossa tabela de tarefas:

```bash
mix ecto.gen.migration create_tasks_table
```

Isso cria um arquivo dentro de `priv/repo/migrations/` (o nome começa com um _timestamp_ — é assim que o Ecto sabe a ordem de aplicação).

**Ação 2:** abra o arquivo gerado e complete a função `change`:

```elixir
defmodule ElixirTodoList.Repo.Migrations.CreateTasksTable do
  use Ecto.Migration

  def change do
    create table(:tasks) do
      add :title, :string
      add :completed, :boolean, default: false

      timestamps() # Adiciona as colunas inserted_at e updated_at
    end
  end
end
```

**Ação 3:** aplique a migration:

```bash
mix ecto.migrate
```

**Saída esperada:**

```
[info] == Running ... ElixirTodoList.Repo.Migrations.CreateTasksTable.change/0 forward
[info] create table tasks
[info] == Migrated ... in 0.0s
```

### 📘 Passo 3.5: Conceitos-Chave — Schema, Changeset e Form

Para conectar o código à tabela recém-criada, precisamos de três peças:

| Conceito        | O que é                                                                  | Analogia                                        |
| --------------- | ------------------------------------------------------------------------ | ----------------------------------------------- |
| **Schema**      | Define a estrutura da tabela e cria uma struct `%Task{}` correspondente. | O "model" do Django ou o ActiveRecord do Rails. |
| **Changeset**   | Um conjunto de regras para validar e transformar dados antes de salvar.  | O `ModelForm` do Django.                        |
| **`to_form/1`** | Converte o changeset para o formato que o LiveView usa no `<.form>`.     | Um _form object_ no MVC.                        |

### 🧩 Passo 3.6: Criar o Schema `Task`

Crie o arquivo `lib/elixir_todo_list/task.ex`:

```elixir
defmodule ElixirTodoList.Task do
  use Ecto.Schema
  import Ecto.Changeset

  # "schema" espelha a tabela "tasks" no banco
  schema "tasks" do
    field :title, :string
    field :completed, :boolean, default: false
    timestamps(type: :utc_datetime)
  end

  # Define como validar os dados antes de salvar
  def changeset(task_struct, attrs) do
    task_struct
    |> cast(attrs, [:title, :completed])
    |> validate_required([:title])
  end
end
```

- `cast/3` → filtra os dados de entrada, aceitando **apenas** os campos listados (`:title`, `:completed`) — proteção contra dados indesejados;
- `validate_required/2` → garante que `:title` não está vazio. É esta linha que fará o formulário exibir _"can't be blank"_ na Fase 4.

### 🧪 Passo 3.7: Testando a Camada de Dados no IEx (sem o LiveView!)

Assim como fizemos no REPL do Clojure, vamos **provar** que a camada de dados funciona **antes** de conectá-la à interface. A versão Elixir do REPL é o **IEx** (Interactive Elixir), e podemos iniciá-lo **com o projeto carregado**:

```bash
iex -S mix
```

No prompt `iex(1)>`, experimente:

```elixir
# Atalhos para digitar menos
iex> alias ElixirTodoList.{Repo, Task}

# 1. O banco está vazio?
iex> Repo.all(Task)
[]

# 2. Crie uma tarefa (validando com o changeset)
iex> %Task{} |> Task.changeset(%{title: "Testar o IEx"}) |> Repo.insert()
{:ok, %ElixirTodoList.Task{id: 1, title: "Testar o IEx", completed: false, ...}}

# 3. E uma inválida? (sem título)
iex> %Task{} |> Task.changeset(%{}) |> Repo.insert()
{:error, #Ecto.Changeset<..., errors: [title: {"can't be blank", ...}], valid?: false>}

# 4. Confira a lista
iex> Repo.all(Task)
[%ElixirTodoList.Task{id: 1, title: "Testar o IEx", ...}]
```

**Momento "Aha!":** repare nas tuplas `{:ok, ...}` e `{:error, changeset}` — são exatamente os dois casos que trataremos com `case` no LiveView, na próxima fase. E a tarefa inválida **não foi salva**: o changeset barrou antes de chegar ao banco.

Saia do IEx com `Ctrl+C` duas vezes.

### 💾 Passo 3.8: Commit (Camada de Dados)

```bash
git add .
git commit -m "Fase 3: Persistência - Configura Ecto, Repo, Migrations e Task Schema"
```

---

**Fim da Fase 3!** 🏁

Agora temos a estrutura completa:

- **O banco de dados** criado e versionado por migrations;
- **O `Repo`** gerenciando conexões e consultas (supervisionado!);
- **O schema `Task`** refletindo nossa tabela, com validação via changeset — tudo testado no IEx.

Falta o passo mais gratificante: trocar o "coração" em memória do `TodoLive` por esse novo, persistente.

---

