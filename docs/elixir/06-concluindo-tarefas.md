# ✅ Fase 6: Refinamento — Concluindo Tarefas (Toggle)

### 🎯 Objetivo

Adicionar um **checkbox** ao lado de cada tarefa. Ao clicar, ele deve marcar a tarefa como **concluída** (ou não concluída) **no banco de dados**, atualizando a interface em tempo real — aplicando ou removendo o estilo "riscado".

### 💡 Por que isso é importante?

Este passo ensina a:

- Criar formulários dinâmicos **dentro de loops** (`:for`);
- Usar `phx-change` para reagir instantaneamente a um checkbox;
- **Atualizar** registros existentes com `Repo.get` + `Repo.update`;
- Reutilizar o `Task.changeset/2` para validação e atualização;
- E a decifrar uma pegadinha clássica dos checkboxes HTML. 🕵️

## 🧩 Passo 6.1: A Interface (adicionando o checkbox)

A maneira mais robusta de lidar com checkboxes no LiveView é criar **um pequeno `<.form>` por tarefa** — assim cada item da lista tem seu próprio estado independente.

Abra `lib/elixir_todo_list_web/live/todo_live.ex`, encontre o loop `<li :for={task <- @tasks}>` no `render/1` e substitua o `<span>` do título por um formulário com checkbox. O bloco da lista **completo** fica assim:

```elixir
        <%!-- A LISTA DE TAREFAS --%>
        <div class="mt-8">
          <ul id="task-list">
            <li
              :for={task <- @tasks}
              id={"task-#{task.id}"}
              class="flex justify-between items-center p-3 border-b"
            >
              <% task_form = Task.changeset(task, %{}) |> to_form() %>

              <.form
                for={task_form}
                phx-change="toggle_complete"
                phx-value-id={task.id}
                class="flex-grow"
              >
                <div class="flex items-center space-x-4">
                  <.input
                    type="checkbox"
                    field={task_form[:completed]}
                    class="flex-shrink-0"
                  />

                  <%!-- Um <label> separado para controle total do estilo --%>
                  <label class={if task.completed, do: "line-through text-gray-500", else: "text-gray-900"}>
                    {task.title}
                  </label>
                </div>
              </.form>

              <.button
                type="button"
                phx-click="delete"
                phx-value-id={task.id}
                class="!p-2 !bg-red-600 hover:!bg-red-700 text-white font-bold rounded-full"
              >
                &times;
              </.button>
            </li>
          </ul>
        </div>
```

### 🧠 Explicação didática

| Elemento                                                 | Função                                                                                                                                              |
| -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `<% task_form = ... %>`                                  | Cria uma variável temporária: o formulário individual **daquela** tarefa (um changeset "vazio" sobre a tarefa existente, convertido com `to_form`). |
| `id={"task-#{task.id}"}`                                 | Um `id` único por `<li>` — ajuda o LiveView a rastrear cada item.                                                                                   |
| `phx-change="toggle_complete"`                           | Evento disparado automaticamente ao marcar/desmarcar o checkbox.                                                                                    |
| `phx-value-id={task.id}`                                 | Envia o **ID da tarefa** junto ao evento.                                                                                                           |
| `<.input type="checkbox" field={task_form[:completed]}>` | O checkbox integrado ao campo `completed` do schema (já renderiza marcado/desmarcado conforme o banco).                                             |
| `<label class={...}>`                                    | Aplica o "riscado" quando a tarefa está concluída.                                                                                                  |

💥 **Erro esperado:** salve o arquivo e clique em um checkbox. Como na Fase 5, o terminal mostrará um erro — o evento `"toggle_complete"` ainda não tem tratador. Já sabemos o remédio.

## 🕵️ Passo 6.2: A Pegadinha do Checkbox (leia antes de codar!)

Antes de escrever o `handle_event`, precisamos entender **o que exatamente chega ao servidor** quando o checkbox muda. E aqui mora uma pegadinha dupla:

**1. HTML puro:** um checkbox **desmarcado não é enviado** em formulários HTML. Se dependêssemos disso, "desmarcar" não geraria dado nenhum.

**2. A solução do Phoenix:** para contornar isso, o componente `<.input type="checkbox">` renderiza, escondido, um segundo campo:

```html
<input type="hidden" name="task[completed]" value="false" /> <input type="checkbox" name="task[completed]" value="true" ... />
```

Graças ao campo oculto, o servidor **sempre** recebe a chave `"completed"`:

- Checkbox **marcado** → `%{"task" => %{"completed" => "true"}}`
- Checkbox **desmarcado** → `%{"task" => %{"completed" => "false"}}`

!!! warning
    **A consequência prática:** como a chave `"completed"` **sempre existe**, testar sua *presença* (por exemplo, com `Map.has_key?`) **não funciona** — daria `true` nos dois casos, e a tarefa, uma vez marcada, **nunca mais desmarcaria**! O teste correto é sobre o **valor**: `task_params["completed"] == "true"`. Guarde essa: é um dos bugs mais sorrateiros do LiveView.


## ⚙️ Passo 6.3: Implementando o `handle_event("toggle_complete", ...)`

Adicione a função **logo abaixo** do `handle_event("delete", ...)`:

```elixir
  @impl true
  def handle_event("toggle_complete", %{"id" => id, "task" => task_params}, socket) do
    # 1. Busca a tarefa correspondente no banco
    task = Repo.get!(Task, id)

    # 2. Lê o novo estado do checkbox
    #    (graças ao campo oculto, "completed" é sempre "true" ou "false")
    completed_status = task_params["completed"] == "true"

    # 3. Cria um changeset de ATUALIZAÇÃO (repare: partimos de 'task',
    #    a struct vinda do banco — não de %Task{} vazio!)
    changeset = Task.changeset(task, %{completed: completed_status})

    # 4. Atualiza o registro no banco de dados
    Repo.update(changeset)

    # 5. Recarrega a lista para atualizar a UI
    socket = assign(socket, tasks: Repo.all(Task))
    {:noreply, socket}
  end
```

### 🧩 O fluxo, passo a passo

1. **Evento recebido** — ao clicar no checkbox, o navegador envia:

   ```elixir
   %{"id" => "1", "task" => %{"completed" => "true"}}  # marcou
   %{"id" => "1", "task" => %{"completed" => "false"}} # desmarcou
   ```

2. **Busca** — `Repo.get!` localiza a tarefa (aqui usamos a versão com `!`: se o ID sumiu, algo está muito errado e preferimos o erro explícito).
3. **Conversão** — `task_params["completed"] == "true"` transforma a _string_ do formulário em um _booleano_ de verdade. (Ecoa a lição do tutorial de Clojure, onde convertíamos o 0/1 do SQLite — formulários e bancos adoram fingir que booleanos são outra coisa!)
4. **Atualização** — `Task.changeset(task, ...)` + `Repo.update` aplicam a mudança. É o mesmo `changeset/2` da criação: como ele parte da struct recebida, serve tanto para `insert` quanto para `update`.
5. **Re-render** — o `assign` atualiza `@tasks`, e o LiveView redesenha o template.

## 🧪 Passo 6.4: Teste Final (o CRUD completo)

Com o servidor rodando, em `http://localhost:4000`:

1. **Create:** adicione duas ou três tarefas. ✅
2. **Read:** F5 — continuam lá. ✅
3. **Update:** marque o checkbox de uma → ela risca imediatamente. **Desmarque** → o risco some (se não sumir, revise o Passo 6.2!). F5 → o estado persiste. ✅
4. **Delete:** clique no "X" de outra → some. F5 → continua fora. ✅
5. **A prova final:** pare o servidor, suba de novo, F5 → **tudo exatamente como você deixou.** ✅

## 💾 Passo 6.5: Commit

```bash
git add .
git commit -m "Fase 6: Implementa conclusão de tarefas (toggle_complete)"
```

---

## 📄 O Código Completo (para conferência)

Se algo não funcionou, compare seu `lib/elixir_todo_list_web/live/todo_live.ex` com o estado final:

```elixir
defmodule ElixirTodoListWeb.TodoLive do
  use ElixirTodoListWeb, :live_view

  alias ElixirTodoList.Repo
  alias ElixirTodoList.Task

  @impl true
  def mount(_params, _session, socket) do
    tasks = Repo.all(Task)
    changeset = Task.changeset(%Task{}, %{})
    form = to_form(changeset)

    socket =
      assign(socket,
        tasks: tasks,
        form: form
      )

    {:ok, socket}
  end

  @impl true
  def handle_event("save_task", %{"task" => task_params}, socket) do
    changeset = Task.changeset(%Task{}, task_params)

    socket_atualizado =
      case Repo.insert(changeset) do
        {:ok, _new_task} ->
          novo_changeset_vazio = Task.changeset(%Task{}, %{})

          socket
          |> assign(:tasks, Repo.all(Task))
          |> assign(:form, to_form(novo_changeset_vazio))
          |> put_flash(:info, "Tarefa salva com sucesso!")

        {:error, failed_changeset} ->
          assign(socket, form: to_form(failed_changeset))
      end

    {:noreply, socket_atualizado}
  end

  @impl true
  def handle_event("delete", %{"id" => id}, socket) do
    task = Repo.get(Task, id)

    if task do
      Repo.delete(task)
    end

    socket =
      socket
      |> assign(:tasks, Repo.all(Task))
      |> put_flash(:info, "Tarefa removida com sucesso!")

    {:noreply, socket}
  end

  @impl true
  def handle_event("toggle_complete", %{"id" => id, "task" => task_params}, socket) do
    task = Repo.get!(Task, id)
    completed_status = task_params["completed"] == "true"
    changeset = Task.changeset(task, %{completed: completed_status})
    Repo.update(changeset)

    socket = assign(socket, tasks: Repo.all(Task))
    {:noreply, socket}
  end

  @impl true
  def render(assigns) do
    ~H"""
    <Layouts.app flash={@flash}>
      <div class="w-full max-w-lg mx-auto mt-12 p-6 bg-white rounded-lg shadow-md">
        <h1 class="text-3xl font-bold mb-6 text-center text-gray-800">
          Minha Lista de Tarefas (com DB!)
        </h1>

        <.form for={@form} id="task-form" phx-submit="save_task">
          <.input
            field={@form[:title]}
            type="text"
            label="Nova Tarefa"
            placeholder="O que precisa ser feito?"
          />
          <.button variant="primary" phx-disable-with="Salvando...">Adicionar Tarefa</.button>
        </.form>

        <div class="mt-8">
          <ul id="task-list">
            <li
              :for={task <- @tasks}
              id={"task-#{task.id}"}
              class="flex justify-between items-center p-3 border-b"
            >
              <% task_form = Task.changeset(task, %{}) |> to_form() %>

              <.form
                for={task_form}
                phx-change="toggle_complete"
                phx-value-id={task.id}
                class="flex-grow"
              >
                <div class="flex items-center space-x-4">
                  <.input
                    type="checkbox"
                    field={task_form[:completed]}
                    class="flex-shrink-0"
                  />

                  <label class={if task.completed, do: "line-through text-gray-500", else: "text-gray-900"}>
                    {task.title}
                  </label>
                </div>
              </.form>

              <.button
                type="button"
                phx-click="delete"
                phx-value-id={task.id}
                class="!p-2 !bg-red-600 hover:!bg-red-700 text-white font-bold rounded-full"
              >
                &times;
              </.button>
            </li>
          </ul>
        </div>
      </div>
    </Layouts.app>
    """
  end
end
```

---

**Fim da Fase 6!** 🏁

Você agora tem um aplicativo **CRUD completo** em Elixir + Phoenix LiveView:

- **Criar** novas tarefas (com validação);
- **Listar** as existentes;
- **Atualizar** (marcar/desmarcar como concluída);
- **Excluir** tarefas.

Tudo com **atualização em tempo real** e sem escrever uma linha de JavaScript. 💪

---

