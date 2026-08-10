# 🗑️ Fase 5: Refinamento — Excluindo Tarefas

### 🎯 Objetivo

Adicionar um **botão "X"** ao lado de cada tarefa para que, ao ser clicado, ela seja **removida permanentemente do banco de dados e da interface**, sem recarregar a página.

### 💡 O que você aprenderá aqui

- Como **disparar eventos de clique** com `phx-click` (sem formulários);
- Como **enviar dados específicos** para o servidor (o ID da tarefa) com `phx-value-*`;
- Como **interagir com o banco** via `Repo.get` e `Repo.delete`;
- E, de quebra, um pouco mais da linguagem de template do LiveView — o **HEEx**.

## 🧱 Passo 5.1: Atualizando a Interface (o botão "X")

Abra o arquivo:

```
lib/elixir_todo_list_web/live/todo_live.ex
```

Na função `render/1`, localize o bloco `<ul>...</ul>` que exibe as tarefas. Dentro do `<li>`, adicione um botão ao lado do título.

_Antes:_

```elixir
<li :for={task <- @tasks} class="flex justify-between items-center p-3 border-b">
  <span class={if task.completed, do: "line-through text-gray-500", else: "text-gray-900"}>
    {task.title}
  </span>
</li>
```

_Depois (com o botão de exclusão):_

```elixir
<li :for={task <- @tasks} class="flex justify-between items-center p-3 border-b">
  <span class={if task.completed, do: "line-through text-gray-500", else: "text-gray-900"}>
    {task.title}
  </span>

  <%!-- NOVO BOTÃO EXCLUIR --%>
  <.button
    type="button"
    phx-click="delete"
    phx-value-id={task.id}
    class="!p-2 !bg-red-600 hover:!bg-red-700 text-white font-bold rounded-full"
  >
    &times; <%!-- Renderiza um "X" --%>
  </.button>
</li>
```

### 🧠 Entendendo o Template (HEEx)

O Phoenix usa o **HEEx** (_HTML + Embedded Elixir Extended_) — parecido com os templates do Django (Jinja), o ERB do Rails —, mas com uma diferença marcante: **tudo é verificado em tempo de compilação**, inclusive atributos e componentes.

| Sintaxe                 | Função                                                          |
| ----------------------- | --------------------------------------------------------------- |
| `{task.title}`          | Interpola um valor Elixir no HTML, com escape seguro.           |
| `<%!-- ... --%>`        | Comentário do HEEx (não vai para o navegador).                  |
| `<.button>`             | Um **componente** do Phoenix — uma função Elixir que gera HTML. |
| `:for={task <- @tasks}` | Estrutura de repetição do HEEx.                                 |

### ⚡ Entendendo os Atributos do Botão

| Atributo                       | Significado                                                                                       |
| ------------------------------ | ------------------------------------------------------------------------------------------------- |
| `phx-click="delete"`           | Ao clicar, envia o evento **`"delete"`** para o servidor (via WebSocket).                         |
| `phx-value-id={task.id}`       | Envia o **ID da tarefa** junto — o servidor o recebe como `%{"id" => "1"}` (repare: **string!**). |
| `class="!p-2 !bg-red-600 ..."` | Classes Tailwind; o `!` força prioridade sobre o estilo padrão do componente.                     |
| `&times;`                      | Entidade HTML do símbolo "×".                                                                     |

### 🧩 Comparando com outros frameworks

| Framework            | Como enviaria o ID no evento?                                                       |
| -------------------- | ----------------------------------------------------------------------------------- |
| **Django**           | Formulário `<form>` ou JavaScript + `fetch()`                                       |
| **Rails**            | `link_to "X", task_path(task), method: :delete, remote: true`                       |
| **Flask**            | Formulário POST + rota `/delete/<id>`                                               |
| **Phoenix LiveView** | Apenas: `phx-click="delete" phx-value-id={task.id}` — o resto viaja pelo WebSocket. |

## 💥 Passo 5.2: Hora do Erro (e da Descoberta)

Salve o arquivo. Os botões "X" aparecem. Agora **clique em um deles** e olhe o terminal:

> ❌ `** (UndefinedFunctionError) ... no function clause matching in ElixirTodoListWeb.TodoLive.handle_event/3` (ou erro similar)

O Phoenix está certo: enviamos o evento `"delete"`, mas **nenhuma cláusula de `handle_event` combina com ele** — o pattern matching não encontrou destino. Igualzinho ao "texto que desaparece" da Fase 2: evento sem tratador derruba (e reinicia) o processo.

## ⚙️ Passo 5.3: Implementando o `handle_event("delete", ...)`

Adicione esta função **logo abaixo** do `handle_event("save_task", ...)`:

```elixir
  @impl true
  def handle_event("delete", %{"id" => id}, socket) do
    # 1. Busca a tarefa correspondente no banco
    #    (se o id não existir, Repo.get retorna nil)
    task = Repo.get(Task, id)

    # 2. Remove a tarefa (apenas se ela existir)
    if task do
      Repo.delete(task)
    end

    # 3. Atualiza a lista de tarefas na tela e avisa o usuário
    socket =
      socket
      |> assign(:tasks, Repo.all(Task))
      |> put_flash(:info, "Tarefa removida com sucesso!")

    # 4. Retorna o novo estado
    {:noreply, socket}
  end
```

### 🧠 Explicando em detalhes

| Etapa                                               | Descrição                                                                                              |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `def handle_event("delete", %{"id" => id}, socket)` | O pattern matching "escuta" o evento `"delete"`; o segundo parâmetro traz o mapa com o `phx-value-id`. |
| `Repo.get(Task, id)`                                | Busca a tarefa pelo ID (o Ecto converte a string para o tipo da chave). Retorna `nil` se não existir.  |
| `if task do ... end`                                | Defesa contra cliques em botões "velhos" (ex: a tarefa já foi removida em outra aba).                  |
| `Repo.delete(task)`                                 | Exclui o registro do banco.                                                                            |
| `assign(socket, :tasks, Repo.all(Task))`            | Recarrega a lista — a UI re-renderiza sem a tarefa excluída.                                           |
| `put_flash(:info, ...)`                             | A mensagem de confirmação — exibida pelo `Layouts.app` que adicionamos na Fase 4.                      |

!!! tip
    **Alternativa mais "estrita":** `Repo.get!(Task, id)` (com `!`) **lança uma exceção** se o ID não existir, em vez de retornar `nil`. É uma convenção do Ecto: funções com `!` falham "alto". Aqui preferimos a versão defensiva, mas você encontrará o `get!` com frequência por aí.


### 🧩 O ciclo completo do evento

1. Usuário clica no botão "X" →
2. O navegador envia `"delete"` com `%{"id" => "3"}` pelo WebSocket →
3. O servidor executa `handle_event("delete", ...)` →
4. O LiveView atualiza o estado (`assign/3`) →
5. O template HEEx é **re-renderizado automaticamente** →
6. A lista aparece na tela **já sem a tarefa** — em tempo real!

## 🔍 Passo 5.4: Testando

1. Salve o arquivo (o servidor recompila sozinho).
2. Acesse `http://localhost:4000`.
3. Crie algumas tarefas e clique no "X" vermelho de uma delas.

✅ A lista se atualiza imediatamente, com o aviso: _"Tarefa removida com sucesso!"_

🔁 Recarregue a página (F5): a tarefa continua excluída — o `DELETE` persistiu no banco.

## 💾 Passo 5.5: Commit

```bash
git add .
git commit -m "Fase 5: Implementa exclusão de tarefas (delete)"
```

---

**Fim da Fase 5!** 🏁

Seu `TodoLive` agora **cria, valida, lista e exclui** tarefas em tempo real — usando apenas Elixir + LiveView, sem JavaScript.

Falta o "U" do CRUD: marcar tarefas como concluídas.

---

