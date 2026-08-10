# 🏃 Fase 1: "Hello World" — Prova de Vida

**Objetivo:** ligar o servidor Phoenix e verificar se tudo está funcionando — o primeiro "sinal de vida" do projeto. Em seguida, substituir a página padrão pelo **nosso** LiveView.

### 🔌 Passo 1.1: Ligar o Servidor

```bash
mix phx.server
```

O servidor será iniciado e exibirá mensagens no terminal. Na primeira execução, o projeto será compilado — pode demorar um pouco.

Abra o navegador e visite: 👉 **http://localhost:4000**

Se tudo deu certo, você verá a **página de boas-vindas do Phoenix** (com o logo e links para a documentação).

!!! warning
    **Aviso comum no Linux:** se aparecer uma mensagem mencionando **inotify-tools**, o *live reload* (atualização automática do navegador quando os arquivos mudam) não está funcionando. Pare o servidor (`Ctrl+C` duas vezes) e instale:

    ```bash
    sudo apt-get install inotify-tools
    ```

    Depois rode `mix phx.server` novamente.


!!! tip
    **Deixe este terminal rodando.** Diferente do tutorial de Clojure (que precisava de dois servidores), aqui **um único processo** cuida de tudo: backend, frontend e a compilação dos assets. Abra um **segundo terminal** para os comandos de Git e Mix daqui em diante.


### 🧭 Passo 1.2: Alterar o Roteador

O Phoenix está exibindo a página padrão (controlada pelo `PageController`). Vamos trocá-la pelo nosso próprio **LiveView**, que será o coração da aplicação.

Abra o arquivo:

```
lib/elixir_todo_list_web/router.ex
```

Encontre o escopo principal e substitua a rota raiz `/`.

_De:_

```elixir
scope "/", ElixirTodoListWeb do
  pipe_through :browser

  get "/", PageController, :home
end
```

_Para:_

```elixir
scope "/", ElixirTodoListWeb do
  pipe_through :browser

  live "/", TodoLive, :index
end
```

A diferença está na palavra-chave **`live`**:

- `get` → responde com uma página renderizada **uma única vez** por um controller tradicional.
- `live` → mantém uma **conexão em tempo real** (WebSocket), capaz de atualizar o conteúdo dinamicamente.

### 💥 Passo 1.3: O Primeiro Erro (e por que ele é bom)

Salve o arquivo e recarregue a página no navegador. Você verá um erro como:

```
** (UndefinedFunctionError) ... ElixirTodoListWeb.TodoLive ... is undefined
   (module ElixirTodoListWeb.TodoLive is not available)
```

Excelente! 🎉 Isso significa que o Phoenix **entendeu a nova rota**, mas não encontrou o módulo `TodoLive`. Ou seja: a configuração está correta — só falta criarmos o módulo. Esse tipo de erro é um ótimo sinal no Phoenix: o compilador está te guiando sobre o que precisa existir.

### 🧱 Passo 1.4: Criar o LiveView

!!! warning
    **Atenção — nomes de módulos no Elixir**

    O Elixir segue uma convenção rígida: o **nome dos módulos** deve corresponder à **estrutura de diretórios** do projeto. Como nosso projeto foi criado na pasta `elixir_todo_list`, o prefixo dos módulos é `ElixirTodoList` (e, para a camada web, `ElixirTodoListWeb`).

    | Diretório                        | Arquivo        | Módulo                       |
    | -------------------------------- | -------------- | ---------------------------- |
    | `lib/elixir_todo_list_web/live/` | `todo_live.ex` | `ElixirTodoListWeb.TodoLive` |

    Se o nome for diferente (por exemplo, `TodoListWeb.TodoLive`), o compilador **não encontrará o módulo** e você verá exatamente o erro do passo anterior — só que dessa vez sem solução à vista. Em resumo: **use o nome do diretório raiz do projeto como base** para os módulos.


Crie o arquivo:

```
lib/elixir_todo_list_web/live/todo_live.ex
```

E adicione o seguinte código:

```elixir
defmodule ElixirTodoListWeb.TodoLive do
  use ElixirTodoListWeb, :live_view

  # mount/3 é o "construtor" do LiveView — chamado quando a página é carregada
  @impl true
  def mount(_params, _session, socket) do
    {:ok, socket}
  end

  # render/1 define o HTML que será exibido
  @impl true
  def render(assigns) do
    ~H"""
    <div class="p-12">
      <h1 class="text-3xl font-bold">Meu Todo List (Hello LiveView!)</h1>
    </div>
    """
  end
end
```

O que está acontecendo aqui:

- `use ElixirTodoListWeb, :live_view` → diz ao Phoenix que este módulo é um **LiveView** (e já traz, de brinde, os componentes de UI e helpers que usaremos nas próximas fases).
- `mount/3` → chamado quando a página é carregada; é o ponto de inicialização do "estado".
- `render/1` → retorna o HTML a ser exibido.
- `~H""" ... """` → é o **HEEx** (HTML + Embedded Elixir), uma versão do HTML com suporte a expressões Elixir — similar ao JSX do React, mas processado **no servidor** e **verificado em tempo de compilação**.

Salve o arquivo e observe: o servidor recompila automaticamente, o erro desaparece e o navegador atualiza sozinho mostrando:

> **Meu Todo List (Hello LiveView!)**

Isso confirma que o LiveView está funcionando: você acabou de renderizar sua primeira página dinâmica em tempo real, **sem escrever uma linha de JavaScript**.

### 💾 Passo 1.5: Commit

No **segundo terminal** (deixe o servidor rodando no primeiro):

```bash
git add .
git commit -m "Fase 1: Prova de Vida - Substitui a rota raiz por TodoLive"
```

Pronto! 🎯 Nosso projeto Phoenix com LiveView está oficialmente "vivo".

---

## 🧠 Pausa Didática — Entendendo `use`, `@impl true`, `mount` e `render`

### 1. O que faz o `use`

A linha:

```elixir
use ElixirTodoListWeb, :live_view
```

é uma forma especial de **trazer comportamentos e configurações** para o módulo atual — uma **injeção de código** feita durante a compilação.

💡 **Em termos simples:** o `use` importa automaticamente todas as funções, macros e configurações necessárias para que o módulo se comporte como um LiveView.

| Linguagem  | Equivalente aproximado              | O que faz                                      |
| ---------- | ----------------------------------- | ---------------------------------------------- |
| **Java**   | `extends BaseView`                  | Herda métodos e atributos de uma classe base   |
| **Python** | `class MyView(BaseView):`           | Cria uma subclasse com comportamento herdado   |
| **Elixir** | `use ElixirTodoListWeb, :live_view` | Injeta código e comportamentos no módulo atual |

➡️ O `use` **não é herança**, mas **geração de código**: ele deixa o módulo pronto para o ecossistema Phoenix. No nosso projeto, ele também já **importa os componentes de UI** (`<.form>`, `<.input>`, `<.button>`) e disponibiliza o alias `Layouts` — vamos usá-los a partir da Fase 4.

### 2. O papel do `@impl true`

O marcador `@impl true` vem antes de funções que **implementam callbacks** de um **comportamento** (_behaviour_). Um _behaviour_ em Elixir é parecido com uma **interface** em linguagens orientadas a objetos: ele define **quais funções um módulo deve implementar**.

O LiveView espera que cada módulo tenha, no mínimo, `mount/3` e `render/1`. Quando você escreve `@impl true`, está dizendo ao compilador: _"esta função é a implementação esperada pelo comportamento do LiveView"_.

| Linguagem  | Anotação equivalente | Finalidade                                                     |
| ---------- | -------------------- | -------------------------------------------------------------- |
| **Java**   | `@Override`          | Indica que o método sobrescreve outro da interface/classe pai  |
| **Elixir** | `@impl true`         | Indica que a função implementa um callback de um comportamento |

Isso ajuda o compilador a verificar se o nome da função, a quantidade de parâmetros e o contrato do framework estão corretos.

### 3. Entendendo `mount/3` e `render/1`

**`mount/3`** — chamado quando o LiveView é carregado pela primeira vez. É como o **construtor** de uma classe: inicializa o estado (socket) e prepara dados.

```elixir
@impl true
def mount(_params, _session, socket) do
  {:ok, assign(socket, tarefas: [])}
end
```

**`render/1`** — gera o HTML da página a partir das variáveis (`assigns`):

```elixir
@impl true
def render(assigns) do
  ~H"""
  <div>
    <h1>Minhas Tarefas</h1>
    <ul>
      <li :for={tarefa <- @tarefas}>{tarefa}</li>
    </ul>
  </div>
  """
end
```

### 4. O paralelo completo

| Conceito     | Em Elixir (Phoenix)               | Analogia em OO            | Finalidade                  |
| ------------ | --------------------------------- | ------------------------- | --------------------------- |
| `use`        | Injeta código e macros            | `extends` + `import`      | Reutilização de lógica base |
| `@impl true` | Declara implementação de callback | `@Override`               | Validação e clareza         |
| `mount/3`    | Inicializa o LiveView             | Construtor da classe      | Configura o estado inicial  |
| `render/1`   | Retorna o HTML dinâmico           | `render()` / `toString()` | Define a interface visual   |

---

**Fim da Fase 1!** 🏁

---

