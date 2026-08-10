# Fase 2: O Backend Funcional (API com Banco em Memória)

**Objetivo:** Construir a lógica de negócios da nossa API. Antes de nos preocuparmos com um banco de dados real, vamos criar um "banco de dados" que vive _apenas na memória_.

### Passo 2.1: O Conceito (Imutabilidade e por que precisamos de `atom`s)

Em muitas linguagens, se você tem uma lista de "todos", você simplesmente "adiciona" (push/append) um novo item, e a lista original é _modificada_.

Em Clojure, as estruturas de dados (como mapas `{}` e vetores `[]`) são **imutáveis**: uma vez criado, um valor **nunca** pode ser alterado.

Vamos ver um exemplo rápido. `conj` é a função para "adicionar" a um vetor:

```clojure
;; 1. Criamos um vetor chamado 'meu-vetor'
user=> (def meu-vetor [1 2])
#'user/meu-vetor

;; 2. "Adicionamos" o número 3 a ele
user=> (conj meu-vetor 3)
[1 2 3]

;; 3. Agora, vamos checar o 'meu-vetor' original
user=> meu-vetor
[1 2]
```

Veja! O `meu-vetor` original **não mudou**. A função `(conj meu-vetor 3)` não _modificou_ o vetor; ela _retornou um novo vetor_ com o item adicionado.

**Isso cria um problema para nós:** se nosso banco fosse `(def todos-db {})`, ele seria um mapa vazio _para sempre_. Como "salvar" um novo todo?

**A Solução: o `atom` (a "caixa" segura)**

Nós colocamos nosso valor imutável (um mapa `{}`) dentro de uma "caixa" de referência: um **`atom`**.

- O **valor dentro da caixa** (o mapa) é imutável.
- O **`atom` é a caixa** em si.
- Nós não _mudamos_ o valor: nós **trocamos** (_swap_) o valor antigo por um valor _totalmente novo_ dentro da caixa.

Para trabalhar com `atom`s, usamos três coisas:

1. **`(atom {...})`**: o **construtor** — cria a caixa com um valor inicial dentro.
2. **`@`** (lê-se "deref"): o **leitor** — "olha dentro" da caixa e vê o valor atual.
3. **`(swap! ...)`**: o **escritor** — troca o conteúdo da caixa aplicando uma função ao valor antigo. Ele é "atômico" (daí o nome): mesmo que 1000 requisições tentem criar um todo ao mesmo tempo, nenhuma escrita é perdida.

### Passo 2.2: O Banco (`db.clj`)

**Ação:** Crie o arquivo `src/todo/backend/db.clj` (ao lado de `core.clj` e `handler.clj`) e cole:

```clojure
(ns todo.backend.db
  "Este namespace gerencia os dados dos 'todos' em memória.")

;; (def) cria uma 'var' global.
;; (atom {}) cria nossa "caixa" (atom) e coloca
;; um mapa imutável vazio {} dentro dela.
(def todos-db (atom {}))
;; Nosso banco terá a forma: {1 {:id 1, :title "..."}, 2 {:id 2, ...}}

;; Uma "caixa" separada para o contador de IDs.
(def next-id (atom 1))

;; --- Nossas Funções de Acesso ao Banco ---

(defn get-all-todos
  "Retorna uma lista com todos os 'todos' no banco."
  []
  ;; @todos-db: "olha dentro" da caixa (lê o valor).
  ;; (vals): pega apenas os valores do mapa (ignora as chaves/IDs).
  (vec (vals @todos-db)))

(defn create-todo
  "Cria um novo 'todo', salva no banco e o retorna."
  [todo-data] ;; ex: {:title "Minha tarefa"}
  (let [;; 1. Lê o ID atual (ex: 1) usando '@'
        id @next-id

        ;; 2. Cria um NOVO mapa imutável com os dados completos
        new-todo (assoc todo-data
                        :id id
                        :completed false
                        :created-at (str (java.time.Instant/now)))]

    ;; 3. (swap!): "troca" o conteúdo da caixa 'todos-db',
    ;;    aplicando 'assoc' ao mapa antigo para criar um novo.
    (swap! todos-db assoc id new-todo)

    ;; 4. Incrementa o contador na caixa 'next-id'.
    (swap! next-id inc)

    ;; 5. Retorna o 'todo' recém-criado.
    new-todo))
```

### O que fizemos?

- **`todos-db` e `next-id`**: dois `atom`s (caixas) para guardar nosso "banco" e nosso contador de IDs.
- **`get-all-todos`**: uma função que _lê_ o atom (`@`).
- **`create-todo`**: uma função que _escreve_ no atom (`swap!`), adicionando um todo e incrementando o ID.

Agora temos a lógica do banco. Mas, como bons engenheiros, não vamos _assumir_ que funciona. Vamos _provar_.

### Passo 2.3: Teste no REPL

**O que é o REPL?**
REPL significa **Read-Eval-Print-Loop** (Ler-Avaliar-Imprimir-Repetir). É um terminal interativo do Clojure: você digita uma expressão, o Clojure a executa e imprime o resultado. É a ferramenta número um do desenvolvimento em Clojure.

**Ação 1: Inicie o REPL**

Na raiz do projeto:

```bash
clj
```

Você verá um prompt `user=>`. Agora você está "dentro" do Clojure.

**Ação 2: Carregue seu namespace**

```clojure
user=> (require '[todo.backend.db :as db])
```

O REPL deve responder `nil`. Isso é **bom**: significa que o Clojure encontrou e leu seu `db.clj` sem erros. Agora as funções públicas estão disponíveis com o prefixo `db/`.

**Ação 3: Teste as funções!**

```clojure
;; 1. Veja se o banco está vazio
user=> (db/get-all-todos)
[]

;; 2. Crie um 'todo'
user=> (db/create-todo {:title "Testar o REPL"})
{:title "Testar o REPL", :id 1, :completed false, :created-at "2026-..."}

;; 3. Veja se foi salvo!
user=> (db/get-all-todos)
[{:title "Testar o REPL", :id 1, :completed false, :created-at "..."}]

;; 4. Crie mais um
user=> (db/create-todo {:title "Aprender Clojure" :description "É divertido"})
{:title "Aprender Clojure", :description "É divertido", :id 2, ...}

;; 5. Veja a lista completa
user=> (db/get-all-todos)
[{:title "Testar o REPL", :id 1, ...}
 {:title "Aprender Clojure", :description "É divertido", :id 2, ...}]
```

**Ação 4 (Opcional): olhe "dentro" das caixas**

```clojure
;; O mapa completo de 'todos'
user=> @db/todos-db
{1 {:title "Testar o REPL", ...}, 2 {:title "Aprender Clojure", ...}}

;; O contador de ID (deve ser 3 agora)
user=> @db/next-id
3
```

### O que fizemos?

Validamos **100%** da lógica de banco sem tocar em servidor, navegador ou rota. Temos confiança total de que o `db.clj` funciona.

**Para sair do REPL:** pressione `Ctrl+D` (ou digite `(System/exit 0)`).

!!! warning
    Não existe um comando `(exit)` no REPL do `clj` — se você tentar, verá um erro `Unable to resolve symbol: exit`. Use `Ctrl+D`.


### Passo 2.4: Atualização Incremental (Handlers)

Nosso `db.clj` está testado. Agora vamos "conectá-lo" ao servidor web em duas etapas: primeiro ensinamos o `handler.clj` a chamar o `db.clj`; depois ensinamos o `core.clj` (o roteador) a chamar os novos handlers.

**Ação 1: Adicionar os `require`s**

Abra `src/todo/backend/handler.clj`. _Substitua_ a declaração do namespace:

```clojure
(ns todo.backend.handler
  "Este namespace define nossas 'funções de resposta' (Handlers).")
```

por esta:

```clojure
(ns todo.backend.handler
  "Este namespace define nossas 'funções de resposta' (Handlers)."
  (:require [todo.backend.db :as db]     ;; <- ADICIONE ISTO
            [clojure.string :as str]))   ;; <- E ISTO (para validação)
```

**Ação 2: Adicionar os novos handlers**

Agora, _abaixo_ da função `hello-handler` existente, adicione estas duas funções. Note como elas são a "ponte" entre o HTTP e o nosso `db.clj`:

```clojure
;; --- Handler para Listar Todos ---
(defn list-todos-handler
  "Handler para a rota GET /api/todos."
  [_request]
  ;; Chama nossa função de banco e a coloca no 'body'
  {:status 200
   :body {:todos (db/get-all-todos)}})

;; --- Handler para Criar um Todo ---
(defn create-todo-handler
  "Handler para a rota POST /api/todos."
  [request]
  ;; :body é um mapa Clojure que o *middleware*
  ;; (que adicionaremos no próximo passo) vai criar para nós
  ;; a partir do JSON que o cliente enviar.
  (let [todo-data (:body request)
        title (:title todo-data)]

    ;; É uma boa prática validar os dados que chegam.
    (if (and title (not (str/blank? title)))
      ;; Sucesso! Os dados são válidos.
      (let [new-todo (db/create-todo todo-data)]
        ;; :status 201 = "Created"
        {:status 201
         :body new-todo})

      ;; Erro de validação.
      {:status 400 ;; "Bad Request"
       :body {:error "O 'título' (title) é obrigatório"}})))
```

**Importante:** o servidor ainda não sabe sobre essas funções. Se você rodar agora, `/api/todos` ainda dará `404`.

### Passo 2.5: Atualizar o `deps.edn` (JSON)

Nosso `create-todo-handler` precisa receber JSON, e o `list-todos-handler` quer _enviar_ JSON. Quem faz essas conversões são os **middlewares** da biblioteca `ring-json`, que ainda não está na nossa "lista de compras".

**Ação:** Abra o `deps.edn` e adicione a linha do `ring/ring-json` ao bloco `:deps`:

```clojure
 :deps
 {org.clojure/clojure {:mvn/version "1.11.1"}

  ;; --- Dependências do Backend (API REST) ---
  ring/ring-core          {:mvn/version "1.12.2"}
  ring/ring-jetty-adapter {:mvn/version "1.12.2"}

  ring/ring-json          {:mvn/version "0.5.1"} ;; <- ADICIONE ESTA LINHA

  metosin/reitit-ring     {:mvn/version "0.7.0"}}
```

!!! tip
    **Regra de ouro:** sempre que o `deps.edn` mudar, o servidor precisa ser **parado e reiniciado** para baixar/carregar a nova dependência. Mudanças no `deps.edn` nunca são aplicadas "a quente".


### Passo 2.6: Atualizar o `core.clj` (Rotas e Middlewares)

**O que é Middleware?**
Pense em middlewares como "assistentes" que "envelopam" seu handler. Eles rodam **antes** e **depois** dele, para fazer tarefas comuns:

- `Request (JSON)` → **Middleware (converte JSON → mapa)** → `Seu Handler`
- `Seu Handler` → **Middleware (converte mapa → JSON)** → `Response (JSON)`

**Ação 1: Adicionar os `require`s de middleware**

No topo do `src/todo/backend/core.clj`:

```clojure
(ns todo.backend.core
  (:require [ring.adapter.jetty :as jetty]
            [reitit.ring :as ring]
            ;; --- ADICIONE ESTAS 3 LINHAS ---
            [ring.middleware.json :refer [wrap-json-response wrap-json-body]]
            [ring.middleware.params :refer [wrap-params]]
            [ring.middleware.keyword-params :refer [wrap-keyword-params]]
            ;; --------------------------------
            [todo.backend.handler :as handler])
  (:gen-class))
```

**Ação 2: Atualizar as rotas (`app-routes`)**

Vamos adicionar `GET /api/todos` e `POST /api/todos`, aninhando todas as rotas sob o prefixo comum `/api`:

```clojure
(def app-routes
  (ring/router
   ;; Aninhamos tudo sob "/api"
   ["/api"
    ["/hello" {:get {:handler handler/hello-handler}}]

    ["/todos"
     {:get  {:handler handler/list-todos-handler}      ;; GET  chama 'list'
      :post {:handler handler/create-todo-handler}}]])) ;; POST chama 'create'
```

**Ação 3: Substituir o `def app` (adicionar os middlewares)**

Nosso `def app` atual é muito simples. Substitua-o por esta versão:

```clojure
(def app
  (ring/ring-handler
   app-routes                     ;; Nossas rotas
   (ring/create-default-handler)  ;; Handler padrão para 404 (Not Found)

   ;; --- A "Linha de Montagem" de Middlewares ---
   ;; A requisição percorre o vetor DE CIMA PARA BAIXO até chegar
   ;; ao handler; a resposta volta DE BAIXO PARA CIMA.
   {:middleware [;; 1. Converte a *resposta* (nosso mapa) em JSON
                 wrap-json-response

                 ;; 2. Converte o *corpo* da requisição (JSON)
                 ;;    em um mapa Clojure e o coloca em :body
                 [wrap-json-body {:keywords? true}]

                 ;; 3. Lê parâmetros de URL (ex: /user?id=1)
                 ;;    e os coloca em :query-params (chaves string)
                 wrap-params

                 ;; 4. Converte as chaves-string dos parâmetros
                 ;;    em keywords ("id" -> :id).
                 ;;    Precisa vir DEPOIS do wrap-params!
                 wrap-keyword-params]}))
```

As funções `start-server` e `-main` no final do arquivo ficam exatamente como estão.

- **Vamos entender melhor o novo `(def app ...)`?**

  Pense no `:middleware` como uma **Linha de Montagem** (um _pipeline_):
  - A **Requisição** entra no topo (item 1 do vetor) e desce até o seu handler.
  - A **Resposta** sai do handler e sobe do fundo (item 4) até o topo.

  Vamos seguir uma requisição `POST /api/todos?debug=true` com o corpo `{"title": "Meu Todo"}`:

  **O Caminho da Requisição (Descendo ⬇️)**
  1. **⬇️ `wrap-json-response`** — só se importa com a _resposta_; na requisição, apenas passa adiante.
  2. **⬇️ `wrap-json-body`** — vê que o `Content-Type` é `application/json`, lê o corpo, converte a string JSON em um mapa Clojure e (graças a `{:keywords? true}`) converte as chaves em keywords. Adiciona à requisição: `{:body {:title "Meu Todo"}}`.
  3. **⬇️ `wrap-params`** — a "estação criadora": olha a URL, vê `?debug=true` e adiciona `{:query-params {"debug" "true"}}` (chaves **string**).
  4. **⬇️ `wrap-keyword-params`** — a "estação conversora": encontra os parâmetros criados no passo anterior e converte as chaves para keywords: `{:query-params {:debug "true"}}`.
  5. **⬇️ Handler** — recebe a requisição "enriquecida":

  ```clojure
  {...
   :body {:title "Meu Todo"}
   :query-params {:debug "true"}}
  ```

  **O Caminho da Resposta (Subindo ⬆️)**
  1. **⬆️ Handler** retorna um mapa Clojure puro: `{:status 201 :body {:id 1, ...}}`.
  2. **⬆️ `wrap-keyword-params`** e **⬆️ `wrap-params`** — não se importam com respostas; passam adiante.
  3. **⬆️ `wrap-json-body`** — só se importa com o _corpo da requisição_; passa adiante.
  4. **⬆️ `wrap-json-response`** — vê que o `:body` é um mapa Clojure, converte-o em uma _string JSON_ e adiciona o header `Content-Type: application/json`.

  É por isso que a **ordem é crucial**: o `wrap-params` precisa **criar** os parâmetros _antes_ que o `wrap-keyword-params` possa convertê-los — por isso, no vetor, `wrap-params` vem **antes** de `wrap-keyword-params` (a requisição percorre o vetor de cima para baixo).

### Passo 2.7: Teste (API Completa com `curl`)

**Ação 1: Reinicie o servidor**

Pare o servidor (`Ctrl+C`, se estiver rodando) e inicie de novo, para carregar a nova dependência e o novo código:

```bash
clj -M:run
```

O `clj` vai baixar o `ring/ring-json` e então: `Servidor iniciado na porta 3000`. Deixe rodando.

**Ação 2: Teste em um novo terminal**

**Teste 1 — Listar todos (deve estar vazio):**

```bash
curl http://localhost:3000/api/todos
```

**Resultado Esperado:** uma string JSON com um array vazio. Isso prova que `list-todos-handler` e `wrap-json-response` funcionam:

```json
{ "todos": [] }
```

**Teste 2 — Criar um todo:**

```bash
curl -X POST http://localhost:3000/api/todos \
     -H "Content-Type: application/json" \
     -d '{"title": "Testar a API", "description": "com curl"}'
```

**Resultado Esperado** (o ID pode variar):

```json
{ "title": "Testar a API", "description": "com curl", "id": 1, "completed": false, "created-at": "..." }
```

**Teste 3 — Listar novamente:**

```bash
curl http://localhost:3000/api/todos
```

**Resultado Esperado** (o todo criado agora aparece):

```json
{ "todos": [{ "title": "Testar a API", "description": "com curl", "id": 1, "completed": false, "created-at": "..." }] }
```

**Teste 4 — Validação (título vazio):**

```bash
curl -X POST http://localhost:3000/api/todos \
     -H "Content-Type: application/json" \
     -d '{"title": ""}'
```

**Resultado Esperado:**

```json
{ "error": "O 'título' (title) é obrigatório" }
```

!!! tip
    **Prefere uma interface gráfica?** Ferramentas como **Insomnia** ou **Postman** fazem o mesmo que o `curl`: crie uma requisição, escolha o método (GET/POST), digite a URL e, no caso do POST, selecione *Body → JSON* e cole o corpo. O resultado deve ser idêntico.


!!! warning
    **Erro comum no Windows:** o `curl` do PowerShell trata aspas de forma diferente. Se o POST falhar com erro de JSON, use o *Prompt de Comando* (cmd), o Git Bash, ou o Insomnia/Postman.


### O que fizemos?

Se você viu esses resultados, **parabéns!** 🥳 Você construiu uma API REST funcional em Clojure, que:

1. Recebe e entende JSON (`POST`);
2. Usa o `atom` para salvar dados na memória;
3. Retorna dados como JSON (`GET`);
4. Valida entradas e responde com códigos HTTP corretos (`201`, `400`).

### Passo 2.8: Git Checkpoint (API Funcional)

**Ação 1:** Pare o servidor (`Ctrl+C`) e verifique:

```bash
git status
```

Ele deve listar `db.clj` (novo), e `deps.edn`, `core.clj`, `handler.clj` (modificados).

**Ação 2:** Prepare e salve:

```bash
git add .
git commit -m "feat: implementa API REST de 'todos' com banco em memória"
```

!!! tip
    Se você usa o **VS Code**, essa operação também pode ser feita pela aba *Source Control* da interface — inclusive a publicação no GitHub ("Publish to GitHub"), que criará o repositório remoto para você. Lembre-se: o repositório da entrega deve ser **público**.


---

**Fim da Fase 2!** 🏁

Nosso backend tem um histórico limpo: setup → Hello World → API REST funcional. Ele está pronto e esperando por requisições. Agora, vamos construir algo para _usá-lo_.

---

