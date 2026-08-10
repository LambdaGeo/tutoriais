# Fase 4: Conectando o Frontend ao Backend

**Objetivo:** Substituir o estado local "de mentira" do frontend por chamadas reais à nossa API — e, no caminho, encontrar (de propósito!) e corrigir o famoso erro de **CORS**.

### Passo 4.1: Adicionar o `core.async` ao `deps.edn`

Para conversar com a API, o frontend fará requisições **assíncronas** (`fetch`). Em ClojureScript, a forma idiomática de lidar com assincronia é a biblioteca `core.async` (com os blocos `go` e o operador `<p!`, que "espera" uma Promise do JavaScript).

**Ação:** Abra o `deps.edn` e adicione o `core.async` ao bloco `:deps`:

```clojure
  ;; --- Dependências do Frontend ---
  thheller/shadow-cljs    {:mvn/version "2.28.23"}
  reagent/reagent         {:mvn/version "2.0.0"}
  org.clojure/core.async  {:mvn/version "1.6.681"} ;; <- ADICIONE ESTA LINHA
```

!!! warning
    **Lembre-se da regra de ouro:** mudou o `deps.edn`, **reinicie** o que estiver rodando. Se o `npx shadow-cljs watch app` estiver ativo, pare-o (`Ctrl+C`) e suba de novo — só assim ele enxerga a nova dependência.


### Passo 4.2: O Primeiro `fetch` — e o Erro CORS (Intencional)

**Objetivo:** vamos tentar buscar a lista de todos do backend. Vamos _esperar_ que isso falhe, para ver _por que_ o CORS existe.

**Ação 1: Adicionar os `require`s de assincronia**

No `ns` do `src/todo/frontend/core.cljs`:

```clojure
(ns todo.frontend.core
  (:require [reagent.core :as r]
            [reagent.dom.client :as rdom]
            [clojure.string :as str]
            ;; --- ADICIONE ESTAS DUAS LINHAS ---
            [cljs.core.async :refer [go]]
            [cljs.core.async.interop :refer-macros [<p!]]))
```

**Ação 2: Adicionar as funções da API**

Abaixo do `defonce app-state`, adicione o `api-url` e as funções `fetch-json` e `get-todos`:

```clojure
;; --- A URL base da nossa API (o backend, porta 3000) ---
(def api-url "http://localhost:3000/api")

;; Função auxiliar: faz o fetch e converte a resposta JSON
;; em dados Clojure (mapas com keywords).
(defn fetch-json [url options]
  (-> (js/fetch url (clj->js options))
      (.then (fn [response]
               (when-not (.-ok response)
                 (throw (js/Error. (str "HTTP error: " (.-status response)))))
               (.json response)))
      ;; Converte o objeto JS em dados Clojure com chaves keyword
      (.then #(js->clj % :keywordize-keys true))))

;; Busca todos os "todos" da API
(defn get-todos []
  (swap! app-state assoc :loading true :error nil)
  (go
    (try
      (let [response (<p! (fetch-json (str api-url "/todos") {:method "GET"}))]
        (swap! app-state assoc :todos (:todos response) :loading false))
      (catch js/Error e
        (swap! app-state assoc :error (.-message e) :loading false)))))
```

!!! tip
    **Como ler o `go` + `<p!`:** o bloco `(go ...)` cria um "processo" assíncrono. Dentro dele, `(<p! promise)` significa "pause aqui até a Promise resolver e me dê o valor". É o equivalente do `async/await` do JavaScript, em ClojureScript.


**Ação 3: Chamar `get-todos` na inicialização**

Encontre a função `init` no final do arquivo e adicione a chamada:

```clojure
(defn ^:export init []
  (println "Frontend inicializado...")
  (let [root (rdom/create-root (js/document.getElementById "app"))]
    (.render root (r/as-element [app])))

  ;; --- ADICIONE ESTA LINHA ---
  ;; Ao carregar a página, busca os todos da API
  (get-todos))
```

**Ação 4: Teste (A Falha Intencional)**

Agora, sim, vamos rodar **ambos** os servidores:

1. **Terminal 1 (Backend):**

   ```bash
   clj -M:run
   ```

   (Deve dizer `Servidor iniciado na porta 3000`. Este servidor ainda _não_ sabe nada sobre CORS.)

2. **Terminal 2 (Frontend):**

   ```bash
   npx shadow-cljs watch app
   ```

   (Espere o `Build completed`.)

3. **Navegador:** abra `http://localhost:8000`.

**Resultado Esperado:** a página carrega, mas a lista de todos **não** aparece.

**A "Lição" (O "Aha!"):**

1. Abra o **Console do Desenvolvedor** (`F12`, aba _Console_).
2. Você verá um **erro vermelho** bem claro:

```
Access to fetch at 'http://localhost:3000/api/todos' from origin
'http://localhost:8000' has been blocked by CORS policy...
```

### O que aprendemos

Você acabou de descobrir o **CORS** (_Cross-Origin Resource Sharing_). O navegador, por segurança, impediu que o `localhost:8000` (o frontend) fizesse uma requisição para o `localhost:3000` (o backend), porque eles são "origens" diferentes (portas diferentes contam como origens diferentes!).

Importante: **quem bloqueia é o navegador**, não o servidor. É por isso que o `curl` funcionava — ele não é um navegador e não aplica a política de CORS. A correção, porém, é feita **no servidor**: é ele quem precisa declarar "eu confio na origem `localhost:8000`".

### Passo 4.3: Corrigir o CORS (no Backend)

**Ação 1: Adicionar a dependência**

Abra o `deps.edn` e adicione o `ring-cors` ao bloco `:deps` (junto às dependências do backend):

```clojure
  ;; --- Dependências do Backend (API REST) ---
  ring/ring-core          {:mvn/version "1.12.2"}
  ring/ring-jetty-adapter {:mvn/version "1.12.2"}
  ring/ring-json          {:mvn/version "0.5.1"}
  metosin/reitit-ring     {:mvn/version "0.7.0"}
  ring-cors/ring-cors     {:mvn/version "0.1.13"} ;; <- ADICIONE ESTA LINHA
```

!!! warning
    Atenção ao nome: é `ring-cors/ring-cors` (grupo e artefato iguais). Escrever `ring/ring-cors` fará o download falhar.


**Ação 2: Adicionar o middleware ao `core.clj`**

Abra `src/todo/backend/core.clj`:

1. **Requeira** o `wrap-cors` no `ns`:

   ```clojure
   (ns todo.backend.core
     (:require [ring.adapter.jetty :as jetty]
               [reitit.ring :as ring]
               [ring.middleware.json :refer [wrap-json-response wrap-json-body]]
               [ring.middleware.params :refer [wrap-params]]
               [ring.middleware.keyword-params :refer [wrap-keyword-params]]
               ;; --- ADICIONE ESTA LINHA ---
               [ring.middleware.cors :refer [wrap-cors]]
               [todo.backend.handler :as handler])
     (:gen-class))
   ```

2. **Adicione o `wrap-cors` como o PRIMEIRO middleware** da "linha de montagem" no `(def app ...)`:

   ```clojure
   (def app
     (ring/ring-handler
      app-routes
      (ring/create-default-handler)
      {:middleware [;; --- ADICIONE ESTE VETOR (o primeiro da lista!) ---
                    [wrap-cors
                     ;; Em quais origens o backend confia:
                     :access-control-allow-origin [#"http://localhost:8000"]
                     ;; Quais métodos HTTP são permitidos:
                     :access-control-allow-methods [:get :post :put :delete]
                     ;; Quais headers o frontend pode enviar:
                     ;; (necessário para o POST com Content-Type: application/json)
                     :access-control-allow-headers ["Content-Type"]]

                    ;; O resto dos middlewares (como antes)...
                    wrap-json-response
                    [wrap-json-body {:keywords? true}]
                    wrap-params
                    wrap-keyword-params]}))
   ```

!!! tip
    **Por que o `wrap-cors` precisa ser o primeiro?** Além das requisições normais, o navegador envia uma requisição extra de "sondagem" chamada **preflight**: um `OPTIONS` perguntando *"posso fazer um POST com o header Content-Type para você?"*. Nós não temos nenhuma rota `OPTIONS` — quem responde a essa pergunta é o próprio `wrap-cors`. Sendo o primeiro (o mais "externo") da linha de montagem, ele intercepta o preflight **antes** de o roteador procurar (e não achar) uma rota.

    E é por isso que declaramos `:access-control-allow-headers ["Content-Type"]`: um `GET` simples não dispara preflight, mas o nosso `POST` com JSON dispara — e o navegador só o libera se o servidor disser explicitamente que aceita esse header. Sem essa linha, você cairia na situação mais confusa possível: _o GET funciona, mas o POST continua bloqueado por CORS_.


**Ação 3: Teste (A Correção)**

1. **Terminal 1 (Backend):** **pare** (`Ctrl+C`) e **reinicie** (`clj -M:run`) — crucial para baixar o `ring-cors`.
2. **Terminal 2 (Frontend):** pode deixar rodando.
3. **Navegador:** volte a `http://localhost:8000` e recarregue (F5).

**Resultado Esperado:** o erro de CORS no console (F12) desapareceu! A página ainda estará vazia (não há todos no banco em memória), mas sem erros. Seu frontend agora pode "falar" com o backend.

### Passo 4.4: Conectar a Criação (`POST`)

O `GET` funciona. Agora vamos fazer o botão "Adicionar" parar de usar a função local (`adicionar-todo-local`) e usar a API de verdade.

**Ação 1: Adicionar a função `create-todo`**

No `src/todo/frontend/core.cljs`, logo **abaixo** da função `get-todos`, adicione:

```clojure
;; --- Cria um "todo" via API (POST) ---
(defn create-todo [todo-data]
  (swap! app-state assoc :loading true :error nil)
  (go
    (try
      (<p! (fetch-json (str api-url "/todos")
                       {:method "POST"
                        :headers {"Content-Type" "application/json"}
                        ;; Converte o mapa Clojure em uma string JSON
                        :body (js/JSON.stringify (clj->js todo-data))}))

      ;; Se o POST funcionou, recarregamos a lista
      (get-todos)
      (catch js/Error e
        (swap! app-state assoc :error (.-message e) :loading false)))))
```

**Ação 2: Modificar o `todo-form`**

Agora temos duas funções de "adicionar": a antiga `adicionar-todo-local` (de mentira) e a nova `create-todo` (da API). Vamos ligar o formulário na nova.

_Mude esta versão:_

```clojure
(defn todo-form []
  [:div.todo-input
   [:input
    {:type "text"
     :placeholder "O que precisa ser feito?"
     :value (:input-text @app-state)
     :on-change #(swap! app-state assoc :input-text (-> % .-target .-value))}]

   [:button
    {:on-click adicionar-todo-local} ;; <-- MUDE AQUI
    "Adicionar (Local)"]])           ;; <-- E AQUI
```

_Para esta versão:_

```clojure
(defn todo-form []
  [:div.todo-input
   [:input
    {:type "text"
     :placeholder "O que precisa ser feito?"
     :value (:input-text @app-state)
     :on-change #(swap! app-state assoc :input-text (-> % .-target .-value))}]

   [:button
    {:on-click (fn []
                 (create-todo {:title (:input-text @app-state)})
                 (swap! app-state assoc :input-text ""))} ;; Limpa o input
    "Adicionar"]])
```

**Ação 3: Limpar o código**

Duas pequenas faxinas para o código não "mentir":

1. **Delete** a função `adicionar-todo-local` — ela não é mais usada. (Como consequência, a chave `:next-id` do `app-state` também ficou sem uso; pode removê-la do `defonce`, deixando `{:input-text "" :todos []}`.)
2. No componente `app`, **atualize o texto**, que ainda dizia que os dados somem no F5:

_Mude:_

```clojure
   [:h1 "Todo App (Somente Frontend)"]
   [:p "Isto é 100% local. Recarregue (F5) para ver os dados sumirem."]
```

_Para:_

```clojure
   [:h1 "Todo App"]
   [:p "Conectado à API. Os dados sobrevivem ao F5!"]
```

!!! tip
    Depois de deletar o `defonce` antigo com `:next-id`, o *hot-reload* pode manter o valor antigo na memória (é justamente o que o `defonce` faz!). Se algo parecer estranho, um simples **F5** recarrega tudo do zero.


### Passo 4.5: Teste Final (Full-Stack)

**Ação 1:** confira que os dois servidores estão de pé:

1. **Terminal 1:** `clj -M:run` → `Servidor iniciado na porta 3000` (com a correção do CORS).
2. **Terminal 2:** `npx shadow-cljs watch app` → `Build completed`.

**Ação 2: Teste no navegador**

1. Abra a URL do **frontend**: `http://localhost:8000`.
2. Confira o console (F12): **sem erros de CORS**. Lista vazia.
3. Digite um todo (ex: "Testar a API completa") e clique em "Adicionar".

**O que acontece agora (a mágica):**

- O `core.cljs` chama `create-todo` (o `POST` para a porta 3000);
- O navegador dispara antes um **preflight** `OPTIONS`, que o `wrap-cors` responde;
- O backend cria o todo no `atom`;
- O `create-todo` chama `get-todos` (o `GET`);
- O `@app-state` é atualizado com a nova lista;
- O Reagent redesenha a UI → **seu todo aparece na lista!**

**Ação 3: O "Aha!" (a persistência da API)**

Faça o teste que falhou na Fase 3: **recarregue a página (F5)**.

- O `init` roda de novo e chama `get-todos`;
- O backend (que **não** foi reiniciado) ainda tem o todo no seu `atom` e o devolve;
- **Resultado:** o todo **continua lá!**

A aplicação agora é _full-stack_: a UI está desacoplada dos dados, e os dados sobrevivem ao recarregamento da página (por enquanto, na memória do backend).

### Passo 4.6: Git Checkpoint

**Ação 1:** pare os servidores e verifique:

```bash
git status
```

Você deve ver: `deps.edn` (core.async + ring-cors), `src/todo/backend/core.clj` (wrap-cors) e `src/todo/frontend/core.cljs` (funções de API + limpeza).

**Ação 2:** prepare e salve:

```bash
git add .
git commit -m "feat: conecta frontend com API do backend (CORS corrigido)"
```

---

**Fim da Fase 4!** 🏁

Temos uma aplicação full-stack funcional, com o histórico salvo no Git.

Mas ela ainda tem uma grande limitação: se você parar e reiniciar o servidor de **backend** (`clj -M:run`), os todos desaparecem — eles vivem em um `atom`, que é memória volátil. Faça o teste, se quiser: pare o backend, suba de novo, dê F5… lista vazia. 😢

A próxima fase resolve isso trocando o `db.clj` por um banco de dados **real**.

---

