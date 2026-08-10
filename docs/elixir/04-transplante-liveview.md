# 🫀 Fase 4: O "Transplante" — Conectando o LiveView ao Banco

Já temos o banco criado, o schema configurado e o `Repo` operacional (Fase 3). Chegou o momento de **conectar tudo ao nosso LiveView**.

Chamamos esta etapa de "transplante" porque substituímos o coração antigo (estado em memória) por um novo (persistência real) — **sem mudar o corpo** (a estrutura mount/eventos/render permanece a mesma).

### 🔍 Visão Geral: o que vai mudar?

**Antes**, nossa aplicação:

- Mantinha as tarefas apenas no **socket** (memória);
- Cada F5 apagava tudo;
- O formulário era um `<form>` "cru", com o estado do campo gerenciado manualmente (`update_form`).

**Agora**:

- O **`Repo`** vira a ponte entre o LiveView e o banco;
- O **schema `Task`** substitui os mapas manuais;
- O **changeset + `to_form/1`** cuidam da validação e da integração com o formulário;
- E as mensagens de sucesso aparecerão via **flash**.

### 1️⃣ Passo 4.1: O Setup do Módulo (aliases)

Abra `lib/elixir_todo_list_web/live/todo_live.ex` e ajuste o **topo** do módulo:

```elixir
defmodule ElixirTodoListWeb.TodoLive do
  use ElixirTodoListWeb, :live_view

  # Atalhos (aliases) para não digitar o nome completo toda hora
  alias ElixirTodoList.Repo
  alias ElixirTodoList.Task
```

- `alias` permite escrever `Repo.all(...)` em vez de `ElixirTodoList.Repo.all(...)`.

!!! tip
    **E os componentes de UI (`<.form>`, `<.input>`, `<.button>`)?** Você **não** precisa importar nada: o `use ElixirTodoListWeb, :live_view` (lá do topo) já importa o `CoreComponents` e disponibiliza o alias `Layouts` para todos os LiveViews. É uma das "injeções" que a Pausa Didática da Fase 1 explicou.


### 2️⃣ Passo 4.2: O "Construtor" — Carregamento Inicial (`mount/3`)

Substitua o `mount/3` por:

```elixir
  @impl true
  def mount(_params, _session, socket) do
    tasks = Repo.all(Task)                     # Lê as tarefas do banco de dados
    changeset = Task.changeset(%Task{}, %{})   # Cria um "molde" vazio
    form = to_form(changeset)                  # Converte para o formulário de UI

    socket =
      assign(socket,
        tasks: tasks,
        form: form
      )

    {:ok, socket}
  end
```

**O que mudou?**

| Antes                                    | Agora                                                        |
| ---------------------------------------- | ------------------------------------------------------------ |
| Lista fixa de tarefas escrita no código. | Tarefas reais do banco, via `Repo.all(Task)`.                |
| Um simples `new_task_title` no estado.   | Um `changeset` convertido em `form` — validável e integrado. |

> 💡 O `mount/3` é como o `get()` de uma view do Django ou o `index()` de um controller Rails: define o estado inicial da página.

### 3️⃣ Passo 4.3: O "Coração" — Salvando no Banco (`handle_event/3`)

Agora, **remova as duas funções `handle_event`** da Fase 2 (`"update_form"` e `"save_task"`) e coloque no lugar esta única função:

```elixir
  @impl true
  def handle_event("save_task", %{"task" => task_params}, socket) do
    # 1. Cria um changeset com os dados do formulário
    changeset = Task.changeset(%Task{}, task_params)

    # 2. Tenta inserir no banco — e trata os DOIS resultados possíveis
    socket_atualizado =
      case Repo.insert(changeset) do
        # 2A. SUCESSO!
        {:ok, _new_task} ->
          novo_changeset_vazio = Task.changeset(%Task{}, %{})

          socket
          |> assign(:tasks, Repo.all(Task))              # Recarrega as tarefas
          |> assign(:form, to_form(novo_changeset_vazio)) # Reseta o formulário
          |> put_flash(:info, "Tarefa salva com sucesso!")

        # 2B. FALHA! (validação — ex: título em branco)
        {:error, failed_changeset} ->
          # Re-atribui o formulário *com os erros* para exibi-los
          assign(socket, form: to_form(failed_changeset))
      end

    # 3. Retorna o socket atualizado
    {:noreply, socket_atualizado}
  end
```

**Mudanças-chave:**

- Antes, adicionávamos a tarefa direto na lista do socket → agora, `Repo.insert(changeset)` **salva no banco**.
- As tuplas `{:ok, ...}` / `{:error, ...}` que vimos no IEx (Fase 3) reaparecem aqui, tratadas pelo `case`.
- `put_flash/3` prepara uma **mensagem amigável** (como o `messages.success()` do Django).

> 💡 Repare que **não existe mais** o `handle_event("update_form", ...)`. Por quê? Ao trocar o `<form>` cru pelo componente `<.form>` com `to_form` (próximo passo), quem passa a controlar o valor do campo é o próprio mecanismo de formulários do Phoenix — não precisamos mais rastrear cada tecla.

!!! warning
    **Por que o `case` precisa "devolver" o socket?** Note que atribuímos o resultado do `case` a `socket_atualizado` e retornamos **ele**. Um erro clássico é chamar `Repo.insert` e esquecer de usar o socket que saiu do `case` — aí a UI nunca reflete a mudança. Em Elixir, dados são imutáveis: o socket "novo" é um **valor retornado**, nunca um efeito colateral.


### 4️⃣ Passo 4.4: O "Desenhista" — Renderização com `<.form>` e `Layouts.app`

Substitua o `render/1` por:

```elixir
  @impl true
  def render(assigns) do
    ~H"""
    <Layouts.app flash={@flash}>
      <div class="w-full max-w-lg mx-auto mt-12 p-6 bg-white rounded-lg shadow-md">
        <h1 class="text-3xl font-bold mb-6 text-center text-gray-800">
          Minha Lista de Tarefas (com DB!)
        </h1>

        <%!-- O formulário agora usa @form --%>
        <.form for={@form} id="task-form" phx-submit="save_task">
          <.input
            field={@form[:title]}
            type="text"
            label="Nova Tarefa"
            placeholder="O que precisa ser feito?"
          />
          <.button variant="primary" phx-disable-with="Salvando...">Adicionar Tarefa</.button>
        </.form>

        <%!-- A LISTA DE TAREFAS --%>
        <div class="mt-8">
          <ul id="task-list">
            <li :for={task <- @tasks} class="flex justify-between items-center p-3 border-b">
              <span class={if task.completed, do: "line-through text-gray-500", else: "text-gray-900"}>
                {task.title}
              </span>
            </li>
          </ul>
        </div>
      </div>
    </Layouts.app>
    """
  end
```

**Destaques:**

- **`<Layouts.app flash={@flash}>`** — este envelope é **essencial**: no Phoenix 1.8, é dentro dele que vivem o cabeçalho padrão da aplicação e o **`flash_group`** — o componente que **exibe** as mensagens do `put_flash`. Sem esse envelope, o `put_flash(:info, "Tarefa salva...")` do passo anterior rodaria... e a mensagem **nunca apareceria na tela**.
- **`<.form for={@form}>`** — o componente de formulário integrado ao changeset. Repare que os campos agora chegam ao servidor "embrulhados": `%{"task" => %{"title" => "..."}}` — por isso o pattern do `handle_event` mudou para `%{"task" => task_params}`.
- **`<.input field={@form[:title]}>`** — o campo integrado: exibe o valor, o label **e as mensagens de erro de validação**, tudo automaticamente.
- **`<.button variant="primary">`** — o botão estilizado do daisyUI; `phx-disable-with` troca o texto enquanto o envio está em andamento.

### 🧪 Passo 4.5: Testando Tudo

Suba o servidor (`mix phx.server`) e acesse `http://localhost:4000`:

1. **Sucesso:** adicione "Tarefa 1" → ela aparece na lista, o formulário limpa e a mensagem _"Tarefa salva com sucesso!"_ surge no topo. ✅
2. **Falha (validação):** clique em "Adicionar Tarefa" com o campo em branco → aparece o erro **"can't be blank"** junto ao campo. ✅
3. **O teste que falhava na Fase 2:** adicione "Tarefa 2" e **recarregue a página (F5)** → as tarefas **continuam lá**. ✅
4. **A prova final:** **pare o servidor** (`Ctrl+C` duas vezes) e suba de novo (`mix phx.server`) → F5 → **tudo continua lá.** Persistência real! 🎉

### 💾 Passo 4.6: Commit

```bash
git add .
git commit -m "Fase 4: Refatora TodoLive para usar Ecto, Repo e to_form()"
```

---

**Fim da Fase 4!** 🏁

Com essa refatoração, nosso app deixou de ser um protótipo volátil e virou uma aplicação completa, com:

- **Banco de dados real** (Ecto + SQLite);
- **Validação automática** via changeset;
- **Formulário dinâmico** com `to_form()` e mensagens de flash funcionando;
- **Interface reativa** com Phoenix LiveView.

### 🔮 Próximos desafios

1. Adicionar o botão de **excluir** (`phx-click="delete"`) → **Fase 5**;
2. Adicionar o **checkbox de conclusão** (`phx-change="toggle_complete"`) → **Fase 6**.

---

