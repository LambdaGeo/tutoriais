# 🧠 Fase 2: Lógica em Memória

**Objetivo:** construir a lógica que permite adicionar tarefas, **sem ainda usar banco de dados**.

Nesta fase, o foco é compreender **como o Phoenix LiveView mantém o estado da interface em tempo real**, usando apenas o **socket** — o elo entre cliente e servidor.

### 🧩 O que é o "estado" no LiveView?

No modelo LiveView, **o estado da aplicação vive no servidor**. Sempre que o usuário interage com a interface (digita, clica, envia um formulário), o navegador envia um pequeno evento via WebSocket. O servidor processa o evento, **atualiza o estado** e devolve o HTML atualizado.

Esse "estado" é armazenado no **socket**, uma estrutura que representa a sessão daquele usuário. Tudo o que a tela precisa saber — tarefas, campos de formulário, filtros — é guardado ali.

```
Socket = "memória viva" do LiveView
assign(socket, chave: valor)  ➜  grava ou atualiza o estado
@chave                        ➜  acessa o valor no render
```

> 💡 **Conexão com o tutorial de Clojure:** o socket cumpre aqui o papel que o `r/atom` cumpria no Reagent — a "caixa" observada cuja mudança redesenha a interface. A diferença: lá a caixa vivia **no navegador**; aqui, ela vive **no servidor**.

## ⚙️ Passo 2.1: O Estado Inicial (`mount/3`)

Quando o usuário abre a página, o LiveView chama `mount/3`. É aqui que criamos o estado inicial — uma mini "memória RAM" para aquele cliente.

Abra o arquivo:

```
lib/elixir_todo_list_web/live/todo_live.ex
```

e substitua o `mount/3` atual por:

```elixir
  # @impl true → indicamos que esta função faz parte do comportamento LiveView
  @impl true
  def mount(_params, _session, socket) do
    # Nossa "memória" inicial: duas tarefas de exemplo
    tasks = [
      %{id: 1, title: "Comprar leite", completed: false},
      %{id: 2, title: "Aprender LiveView", completed: true}
    ]

    # assign/2 coloca dados no socket (nosso estado de interface)
    socket =
      assign(socket,
        tasks: tasks,
        new_task_title: "" # o campo de texto começa vazio
      )

    # {:ok, socket} → retorna o estado inicial ao LiveView
    {:ok, socket}
  end
```

## 🎨 Passo 2.2: Mostrando o Estado (`render/1`)

A função `render/1` é o "desenhista" da interface: recebe os dados do socket (via `assigns`) e devolve o HTML dinâmico.

Substitua o `render/1` por:

```elixir
  @impl true
  def render(assigns) do
    ~H"""
    <div class="w-full max-w-lg mx-auto mt-12 p-6 bg-white rounded-lg shadow-md">
      <h1 class="text-3xl font-bold mb-6 text-center text-gray-800">
        Minha Lista de Tarefas
      </h1>

      <%!-- FORMULÁRIO DE ENTRADA --%>
      <form phx-submit="save_task" phx-change="update_form" class="flex gap-2 mb-6">
        <input
          type="text"
          name="title"
          value={@new_task_title}
          placeholder="O que precisa ser feito?"
          class="flex-grow p-2 border rounded"
          autofocus
        />
        <button type="submit" class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600">
          Adicionar
        </button>
      </form>

      <%!-- LISTA DE TAREFAS --%>
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
    """
  end
```

### 🧭 Entendendo os atributos "mágicos"

| Atributo                   | Função                                                        |
| -------------------------- | ------------------------------------------------------------- |
| `phx-submit="save_task"`   | Dispara o evento `"save_task"` quando o formulário é enviado. |
| `phx-change="update_form"` | Dispara o evento `"update_form"` a cada tecla digitada.       |
| `value={@new_task_title}`  | Liga o campo de texto ao estado do socket.                    |
| `:for={task <- @tasks}`    | Repete o `<li>` para cada tarefa (o "for" do HEEx).           |

## ⚠️ Passo 2.3: O "Aha!" — o texto que desaparece

Salve o arquivo e recarregue a página. Você verá o formulário e as duas tarefas de exemplo.

Agora **digite algo no campo**. Percebeu? **O texto desaparece a cada letra!**

Isso acontece porque o evento `"update_form"` é disparado, mas **ainda não existe uma função para tratá-lo** (`handle_event`). O processo LiveView falha, o supervisor o **reinicia automaticamente** (é assim que a BEAM lida com erros!), e o `mount/3` roda de novo — limpando o campo. Dê uma olhada no terminal do servidor: o erro está registrado lá.

Essa é uma lição dupla: (1) todo evento `phx-*` precisa de um `handle_event` correspondente; (2) quando um processo LiveView "morre", ele renasce do estado inicial.

## 🧩 Passo 2.4: Tratando a Digitação (`update_form`)

Vamos criar a função que captura o evento `"update_form"`. Adicione **entre o `mount/3` e o `render/1`**:

```elixir
  # Captura o evento de digitação no campo
  @impl true
  def handle_event("update_form", %{"title" => new_title}, socket) do
    # Atualiza o valor do campo no estado
    socket = assign(socket, new_task_title: new_title)
    {:noreply, socket} # retorna o socket atualizado, sem recarregar a página
  end
```

Salve e teste: agora o texto digitado **permanece**. O estado está sendo atualizado **a cada tecla** — e o LiveView reflete isso no HTML.

## 🧱 Passo 2.5: Tratando o Envio (`save_task`)

O próximo passo é realmente **adicionar a nova tarefa** à lista quando o usuário clicar em "Adicionar". Adicione esta função **logo abaixo da anterior**:

```elixir
  # Captura o evento de envio do formulário
  @impl true
  def handle_event("save_task", %{"title" => title}, socket) do
    if String.trim(title) != "" do
      # Cria uma nova tarefa "em memória"
      new_task = %{
        id: System.unique_integer([:positive]),
        title: title,
        completed: false
      }

      # Atualiza a lista de tarefas e limpa o campo
      socket =
        socket
        |> update(:tasks, fn tasks -> tasks ++ [new_task] end)
        |> assign(:new_task_title, "")

      {:noreply, socket}
    else
      # Ignora caso o campo esteja vazio
      {:noreply, socket}
    end
  end
```

| Função                    | Explicação                                                                     |
| ------------------------- | ------------------------------------------------------------------------------ |
| `System.unique_integer/1` | Gera um ID único (suficiente para uso "em memória").                           |
| `update/3`                | Atualiza um valor existente no socket aplicando uma função (a lista `:tasks`). |
| `assign/3`                | Redefine `new_task_title` para limpar o input.                                 |

## 🎉 Resultado

Você tem um **Todo List funcional**, mesmo sem banco de dados! Adicione algumas tarefas e veja a lista crescer instantaneamente.

**O teste da verdade:** agora **recarregue a página (F5)**. As tarefas que você adicionou sumiram — só as duas de exemplo voltaram. E se você reiniciar o servidor, o mesmo acontece.

Isso é esperado: as tarefas viviam **apenas na memória daquele socket**. Cada F5 abre uma conexão nova, e o `mount/3` recomeça do zero. Você acabou de ganhar a **motivação** para a Fase 3: persistência de verdade.

O que aprendemos — o **coração do LiveView**:

- O **estado** mora no **socket**;
- Cada **evento** é tratado por um `handle_event/3`;
- O **`render/1`** reflete o estado atual em HTML, sem recarregar a página.

### 💾 Passo 2.6: Commit

```bash
git add .
git commit -m "Fase 2: Lógica em Memória - Implementa adição de tarefas"
```

---

## 🧠 Pausa Didática — Entendendo o Estilo Elixir

Antes de continuar, vamos fazer uma pausa estratégica para compreender **a estrutura e o estilo da linguagem** que aparecem no nosso código. Mesmo com poucas linhas, o `todo_live.ex` já demonstra vários conceitos centrais: módulos, funções, imutabilidade, pattern matching e o pipe.

Este é o estado atual do nosso arquivo (confira se o seu está igual):

```elixir
defmodule ElixirTodoListWeb.TodoLive do
  use ElixirTodoListWeb, :live_view

  @impl true
  def mount(_params, _session, socket) do
    tasks = [
      %{id: 1, title: "Comprar leite", completed: false},
      %{id: 2, title: "Aprender LiveView", completed: true}
    ]

    socket =
      assign(socket,
        tasks: tasks,
        new_task_title: ""
      )

    {:ok, socket}
  end

  @impl true
  def handle_event("update_form", %{"title" => new_title}, socket) do
    socket = assign(socket, new_task_title: new_title)
    {:noreply, socket}
  end

  @impl true
  def handle_event("save_task", %{"title" => title}, socket) do
    if String.trim(title) != "" do
      new_task = %{
        id: System.unique_integer([:positive]),
        title: title,
        completed: false
      }

      socket =
        socket
        |> update(:tasks, fn tasks -> tasks ++ [new_task] end)
        |> assign(:new_task_title, "")

      {:noreply, socket}
    else
      {:noreply, socket}
    end
  end

  @impl true
  def render(assigns) do
    ~H"""
    ... (o HTML completo do Passo 2.2) ...
    """
  end
end
```

### 1️⃣ Módulos — a "caixa" do código

Em Elixir, tudo vive dentro de um **módulo** (`defmodule`). Um módulo é como uma caixa que organiza funções relacionadas — parecido com uma _classe_, mas com uma diferença fundamental: 👉 **módulos não possuem estado interno nem herança**. Eles apenas **agrupam funções**. O estado vive em estruturas como o **socket** ou em **processos** — nunca no módulo em si.

### 2️⃣ Funções e Aridade

Funções são definidas com `def` (públicas) ou `defp` (privadas). O número após a barra é a **aridade** — quantos parâmetros a função recebe:

- `mount/3` → 3 parâmetros;
- `render/1` → 1 parâmetro;
- `handle_event/3` → 3 parâmetros.

Em Elixir, a aridade **faz parte do nome da função**: `handle_event/2` e `handle_event/3` são funções _diferentes_, e não sobrecargas como em Java ou C#.

### 3️⃣ Pattern Matching — a escolha automática

Observe:

```elixir
def handle_event("update_form", %{"title" => new_title}, socket) do ... end
def handle_event("save_task",   %{"title" => title},     socket) do ... end
```

Temos **duas funções com o mesmo nome e aridade**, e o Elixir **decide qual chamar** pelo **padrão dos argumentos**: se o primeiro argumento for `"update_form"`, executa a primeira; se for `"save_task"`, a segunda. Isso é o **pattern matching** — um dos pilares do paradigma funcional.

Em Python, escreveríamos:

```python
def handle_event(event_name, payload, socket):
    if event_name == "update_form":
        ...
    elif event_name == "save_task":
        ...
```

Em Elixir, **a própria definição da função já é o "if"**.

### 4️⃣ Tuplas — a maneira Elixir de retornar

Em vez de retornar valores "crus", o Elixir usa **tuplas etiquetadas**:

```elixir
{:ok, socket}
{:noreply, socket}
{:error, "algo deu errado"}
```

A primeira posição é sempre um **átomo** (`:ok`, `:noreply`, `:error`) descrevendo o status. Essa convenção vem do Erlang e facilita o controle de fluxo entre processos.

### 5️⃣ O Socket — nosso "estado vivo"

```elixir
socket = assign(socket, tasks: tasks, new_task_title: "")
```

- O `socket` é a "caixa de estado" daquela sessão;
- `assign` **não muta** a caixa: cria uma **nova versão** dela (imutabilidade!);
- Cada atualização do socket dispara uma **nova renderização** automática.

### 6️⃣ O Operador Pipe (`|>`) — elegância funcional

```elixir
socket
|> update(:tasks, fn tasks -> tasks ++ [new_task] end)
|> assign(:new_task_title, "")
```

Lê-se: _"pegue o socket, atualize as tarefas e depois limpe o campo de texto"_. O pipe passa o resultado de cada linha como **primeiro argumento** da próxima, eliminando variáveis intermediárias — como uma receita passo a passo.

### 📘 Recapitulando

| Conceito            | Significado                                 | Analogia                 |
| ------------------- | ------------------------------------------- | ------------------------ |
| `defmodule`         | Agrupa funções relacionadas                 | Classe sem estado        |
| `def` / `defp`      | Funções públicas / privadas                 | Métodos                  |
| Aridade             | Nº de parâmetros (parte do nome)            | —                        |
| Pattern Matching    | Escolhe a função pelo padrão dos argumentos | Substitui `if`/`switch`  |
| Tuplas `{:ok, ...}` | Retornos estruturados                       | `return (status, valor)` |
| `\|>` (pipe)        | Encadeia transformações                     | Fluent interface         |
| `socket`            | Estado imutável da interface                | Store/State no React     |

---

**Fim da Fase 2!** 🏁

Agora estamos prontos para conectar nossa lógica em memória a um **banco de dados real com Ecto**.

---

