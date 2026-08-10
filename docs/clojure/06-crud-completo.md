# 🏆 Fase 6: CRUD Completo — "Marcar como Feito", "Deletar" e o Visual Final

Você tem uma aplicação **C**reate/**R**ead funcional e persistente. Agora vamos completar o CRUD:

- o **U** (Update): marcar um todo como feito/não feito (_toggle_);
- o **D** (Delete): remover um todo.

Você verá que o padrão é _exatamente_ o mesmo que usamos até aqui, repetido duas vezes: **função no `db.clj` → handler no `handler.clj` → rota no `core.clj` → função de API e evento no `core.cljs`**.

---

## ⚙️ Parte 1: O Toggle (Update) no Backend

### Passo 6.1: A Lógica no `db.clj`

Precisamos de uma função que receba um `id` e "vire" o valor de `completed` no banco.

**Ação:** Abra `src/todo/backend/db.clj` e adicione no final:

```clojure
(defn toggle-todo!
  "Alterna o status 'completed' de um todo no banco."
  [id]
  ;; (1 - completed) é um truque SQL para inverter: 0 -> 1 e 1 -> 0.
  ;; RETURNING * devolve a linha já atualizada (ou nada, se o id não existir).
  (jdbc/execute-one! db-spec ["
    UPDATE todos
    SET completed = (1 - completed)
    WHERE id = ?
    RETURNING *"
    id]))
```

!!! tip
    O `!` no final de `toggle-todo!` é uma convenção Clojure: sinaliza que a função tem *efeitos colaterais* (ela muda o mundo — neste caso, o banco). Já vínhamos usando isso no `initialize-database!` e no `swap!`.


### Passo 6.2: O Handler no `handler.clj`

**Ação:** Abra `src/todo/backend/handler.clj` e adicione no final:

```clojure
;; --- Handler para Alternar (toggle) um Todo ---
(defn toggle-todo-handler
  "Handler para a rota POST /api/todos/:id/toggle."
  [request]
  ;; O Reitit coloca os parâmetros da URL (o :id de "/todos/:id/toggle")
  ;; em :path-params, sempre como STRINGS. Convertemos para inteiro.
  (let [id (Integer/parseInt (get-in request [:path-params :id]))]
    ;; execute-one! retorna nil se nenhuma linha foi afetada
    ;; (id inexistente) -> respondemos 404.
    (if-let [updated-todo (db/toggle-todo! id)]
      {:status 200 :body updated-todo}
      {:status 404 :body {:error "Todo não encontrado"}})))
```

### Passo 6.3: A Rota no `core.clj`

**Ação:** Abra `src/todo/backend/core.clj` e adicione a nova rota. Para não haver dúvida de **onde** ela entra, aqui está o `app-routes` **completo** como deve ficar:

```clojure
(def app-routes
  (ring/router
   ["/api"
    ["/hello" {:get {:handler handler/hello-handler}}]

    ["/todos"
     {:get  {:handler handler/list-todos-handler}
      :post {:handler handler/create-todo-handler}}]

    ;; --- NOVA ROTA ---
    ["/todos/:id/toggle"
     {:post {:handler handler/toggle-todo-handler}}]]))
```

**✅ Backend do toggle concluído!** Pare e **reinicie o servidor de backend** (`clj -M:run`).

**Teste rápido com `curl`** (supondo que exista um todo de id 1):

```bash
curl -X POST http://localhost:3000/api/todos/1/toggle
```

**Resultado Esperado:** o todo com `"completed":1`. Rode de novo e ele volta para `0`. E um id inexistente:

```bash
curl -X POST http://localhost:3000/api/todos/999/toggle
```

deve responder `{"error":"Todo não encontrado"}`.

---

## ⚡️ Parte 2: O Toggle (Update) no Frontend

### Passo 6.4: A Função de API no `core.cljs`

**Ação:** Abra `src/todo/frontend/core.cljs` e adicione esta função junto às outras (`get-todos`, `create-todo`):

```clojure
(defn toggle-todo
  "Chama a API para alternar o status de um todo."
  [id]
  (go
    (try
      (<p! (fetch-json (str api-url "/todos/" id "/toggle")
                       {:method "POST"}))
      ;; Se funcionou, recarrega a lista inteira
      (get-todos)
      (catch js/Error e
        (swap! app-state assoc :error (.-message e) :loading false)))))
```

### Passo 6.5: O Checkbox no `todo-list` (e a Conversão 0/1)

**Atenção: por que `(not= 0 ...)` é necessário?**

O SQLite armazena `true` como o número **1** e `false` como **0**. Mas o atributo `checked` de um checkbox HTML só aceita booleanos de verdade (`true`/`false`).

- Se usarmos apenas `:checked (:todos/completed todo)`, o valor seria `1` ou `0`.
- A expressão `(not= 0 (:todos/completed todo))` **converte explicitamente**: `1` → `true` e `0` → `false`.

**Ação:** Modifique a função `todo-list` do `core.cljs`. _Mude esta versão:_

```clojure
(defn todo-list []
  [:ul.todo-list
   (for [todo (:todos @app-state)]
     ^{:key (:todos/id todo)}
     [:li.todo-item
      (:todos/title todo)])])
```

_Para esta versão:_

```clojure
(defn todo-list []
  [:ul.todo-list
   (for [todo (:todos @app-state)]
     ^{:key (:todos/id todo)}

     ;; 1. Classe CSS 'completed' quando o status for 1
     [:li.todo-item {:class (when (= 1 (:todos/completed todo)) "completed")}

      ;; 2. O Checkbox
      [:input.todo-checkbox
       {:type "checkbox"
        ;; 3. A CONVERSÃO: 0/1 -> booleano de verdade
        :checked (not= 0 (:todos/completed todo))
        ;; 4. O evento chama nossa nova função de API
        :on-change #(toggle-todo (:todos/id todo))}]

      ;; 5. O título (agora dentro de um span, para o CSS)
      [:span.todo-title (:todos/title todo)]])])
```

**Teste intermediário:**

1. Com os dois servidores rodando, vá a `http://localhost:8000`.
2. Adicione um todo e **clique no checkbox**. Ele deve marcar.
3. Recarregue (F5): o item **continua marcado** — o `UPDATE` persistiu! 🎉

---

## 🗑️ Parte 3: O Delete — Backend

O padrão se repete pela última vez. Você já sabe a receita: `db` → `handler` → `rota` → `frontend`.

### Passo 6.6: A Lógica no `db.clj`

**Ação:** Adicione ao final de `src/todo/backend/db.clj`:

```clojure
(defn delete-todo!
  "Remove um todo do banco. Retorna o todo removido, ou nil se não existir."
  [id]
  ;; RETURNING * aqui nos devolve a linha que acabou de ser apagada.
  ;; É perfeito para o handler saber se o id existia (nil = não existia).
  (jdbc/execute-one! db-spec ["
    DELETE FROM todos
    WHERE id = ?
    RETURNING *"
    id]))
```

### Passo 6.7: O Handler no `handler.clj`

**Ação:** Adicione ao final de `src/todo/backend/handler.clj`:

```clojure
;; --- Handler para Deletar um Todo ---
(defn delete-todo-handler
  "Handler para a rota DELETE /api/todos/:id."
  [request]
  (let [id (Integer/parseInt (get-in request [:path-params :id]))]
    (if-let [deleted-todo (db/delete-todo! id)]
      {:status 200 :body deleted-todo}
      {:status 404 :body {:error "Todo não encontrado"}})))
```

(Compare com o `toggle-todo-handler`: é o mesmo esqueleto. Isso não é preguiça — é **arquitetura**. Padrões repetíveis são fáceis de escrever, ler e testar.)

### Passo 6.8: A Rota no `core.clj`

**Ação:** O `app-routes` **completo e final** fica assim:

```clojure
(def app-routes
  (ring/router
   ["/api"
    ["/hello" {:get {:handler handler/hello-handler}}]

    ["/todos"
     {:get  {:handler handler/list-todos-handler}
      :post {:handler handler/create-todo-handler}}]

    ["/todos/:id/toggle"
     {:post {:handler handler/toggle-todo-handler}}]

    ;; --- NOVA ROTA ---
    ["/todos/:id"
     {:delete {:handler handler/delete-todo-handler}}]]))
```

!!! tip
    Repare que usamos o **método HTTP** `DELETE` na URL "canônica" do recurso (`/todos/:id`), em vez de inventar uma URL como `/todos/:id/delete`. Essa é a convenção REST: a URL identifica *o recurso*, e o método diz *o que fazer* com ele. (E lembre-se: lá na Fase 4, nosso `wrap-cors` já incluiu `:delete` nos métodos permitidos — por isso o navegador vai deixar essa chamada passar.)


**✅ Backend completo!** Pare e **reinicie** o backend (`clj -M:run`), e teste com `curl`:

```bash
curl -X DELETE http://localhost:3000/api/todos/1
```

**Resultado Esperado:** o JSON do todo removido. Chame de novo com o mesmo id: `{"error":"Todo não encontrado"}` — ele já foi.

---

## 🗑️ Parte 4: O Delete — Frontend

### Passo 6.9: A Função de API e o Botão

**Ação 1:** No `src/todo/frontend/core.cljs`, adicione junto às outras funções de API:

```clojure
(defn delete-todo
  "Chama a API para remover um todo."
  [id]
  (go
    (try
      (<p! (fetch-json (str api-url "/todos/" id)
                       {:method "DELETE"}))
      ;; Se funcionou, recarrega a lista
      (get-todos)
      (catch js/Error e
        (swap! app-state assoc :error (.-message e) :loading false)))))
```

**Ação 2:** Adicione o botão de deletar ao `todo-list`. A versão **final** do componente:

```clojure
(defn todo-list []
  [:ul.todo-list
   (for [todo (:todos @app-state)]
     ^{:key (:todos/id todo)}
     [:li.todo-item {:class (when (= 1 (:todos/completed todo)) "completed")}

      [:input.todo-checkbox
       {:type "checkbox"
        :checked (not= 0 (:todos/completed todo))
        :on-change #(toggle-todo (:todos/id todo))}]

      [:span.todo-title (:todos/title todo)]

      ;; --- O BOTÃO DE DELETAR (agora de verdade!) ---
      [:button.delete-btn
       {:on-click #(delete-todo (:todos/id todo))}
       "X"]])])
```

**Teste intermediário:** adicione alguns todos e clique no "X" de um deles. Ele some da lista. Recarregue (F5): continua fora — o `DELETE` persistiu.

!!! tip
    **Extensão opcional:** quer pedir confirmação antes de apagar? Troque o `:on-click` por:

    ```clojure
    {:on-click #(when (js/confirm "Remover esta tarefa?")
                  (delete-todo (:todos/id todo)))}
    ```


---

## 🎨 Parte 5: O Visual Final (CSS)

Nossa aplicação funciona, mas está com cara de 1996. Vamos dar um banho de loja.

**Ação:** Abra `resources/public/index.html` e adicione o bloco `<style>` dentro do `<head>` (depois do `<title>`):

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Todo App (CLJS)</title>
    <style>
      body {
        font-family: Arial, sans-serif;
        background-color: #f5f5f5;
        display: flex;
        justify-content: center;
        padding: 50px 20px;
      }
      .todo-app {
        width: 100%;
        max-width: 600px;
        background: white;
        padding: 25px;
        border-radius: 8px;
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
      }
      h1 {
        color: #333;
        text-align: center;
        margin-bottom: 25px;
      }
      .todo-input {
        display: flex;
        gap: 10px;
        margin-bottom: 20px;
      }
      .todo-input input[type="text"] {
        flex: 1;
        padding: 10px;
        border: 1px solid #ddd;
        border-radius: 4px;
        font-size: 16px;
      }
      .todo-list {
        list-style: none;
        padding: 0;
      }
      .todo-item {
        display: flex;
        align-items: center;
        padding: 12px;
        margin-bottom: 8px;
        background-color: #f8f9fa;
        border-radius: 4px;
        transition: background-color 0.3s;
      }
      /* O título ocupa todo o espaço entre o checkbox e o botão X */
      .todo-title {
        flex: 1;
      }
      /* O botão "Adicionar" */
      button {
        padding: 10px 15px;
        background-color: #007bff;
        color: white;
        border: none;
        border-radius: 4px;
        cursor: pointer;
        font-size: 16px;
        transition: background-color 0.2s;
      }
      button:hover {
        background-color: #0056b3;
      }
      /* O botão de deletar (X) */
      .delete-btn {
        background-color: #dc3545;
        font-size: 12px;
        padding: 5px 10px;
      }
      .delete-btn:hover {
        background-color: #c82333;
      }
      /* Item concluído */
      .completed {
        opacity: 0.6;
        background-color: #f1f1f1;
        text-decoration: line-through;
      }
      .todo-checkbox {
        margin-right: 10px;
        cursor: pointer;
      }
    </style>
  </head>
  <body>
    <div id="app">Carregando...</div>

    <script src="/js/main.js"></script>
  </body>
</html>
```

!!! tip
    Mudou o `index.html` e não viu diferença? O servidor de desenvolvimento às vezes mantém o HTML em cache — um **F5** (ou Ctrl+F5) resolve.


---

## 🎉 Teste Final (o CRUD completo)

Com os dois servidores rodando, em `http://localhost:8000`:

1. **Create:** adicione dois ou três todos. ✅
2. **Read:** recarregue (F5) — todos continuam lá. ✅
3. **Update:** clique no checkbox de um deles — ele fica riscado e opaco. F5 — continua marcado. ✅
4. **Delete:** clique no "X" de outro — ele some. F5 — continua fora. ✅
5. **A prova final:** pare o backend (`Ctrl+C`), reinicie (`clj -M:run`), F5 no navegador — **tudo exatamente como você deixou.** ✅

Se os cinco passaram, você tem um CRUD full-stack completo e persistente.

## Passo 6.10: Git Checkpoint

**Ação:**

```bash
git add .
git commit -m "feat(crud): implementa funcionalidades de toggle e delete"
```

---

**Fim da Fase 6!** 🏁

Você finalizou o desenvolvimento de uma aplicação full-stack completa, moderna e robusta:

- **Backend:** Clojure, Ring e Reitit expondo uma API REST completa (CRUD).
- **Banco de Dados:** persistência real com SQLite e `next.jdbc`.
- **Frontend:** ClojureScript e Reagent (com o padrão moderno do React) para uma UI reativa.
- **Prática profissional:** `git` incremental, `npm`, depuração de CORS (incluindo _preflight_), dados assíncronos (`go`/`<p!`) e o uso correto de `atom` vs. `r/atom`.

Falta só uma coisa para o projeto estar pronto para o mundo (e para a entrega): a **documentação**.

---

