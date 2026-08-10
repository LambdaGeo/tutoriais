# Fase 5: Persistência Real (Banco de Dados)

**Objetivo:** Substituir nosso `db.clj` "de mentira" (o `atom`) por um `db.clj` "de verdade", que usa um banco de dados **SQLite**.

**Por que SQLite?** É o banco mais simples de usar: não requer um servidor separado (como o PostgreSQL) e armazena o banco inteiro em um único arquivo (`prod.db`) na pasta do projeto.

**Nossa "Cirurgia":**
A parte mais legal desta fase é o quão pouco código vamos mudar. Graças à forma como estruturamos o aplicativo, **só precisamos mudar o `db.clj`**.

Os arquivos `handler.clj` e `core.clj` não se importam com _como_ os dados são armazenados; eles apenas chamam funções como `(db/get-all-todos)`. Vamos reescrever o _interior_ dessas funções para falar SQL em vez de mexer em um `atom`.

### Passo 5.1: Adicionar as Dependências do Banco

- **`next.jdbc`**: a biblioteca moderna de Clojure para "falar" com bancos SQL.
- **`org.xerial/sqlite-jdbc`**: o "driver" que permite ao `next.jdbc` se comunicar especificamente com o SQLite.

**Ação:** Abra o `deps.edn` e adicione estas duas linhas ao bloco `:deps`:

```clojure
  ;; --- Banco de Dados ---
  ;; ADICIONE ESTAS DUAS LINHAS:
  seancorfield/next.jdbc  {:mvn/version "1.2.659"}
  org.xerial/sqlite-jdbc  {:mvn/version "3.45.3.0"}
```

### Passo 5.2: Entendendo o `next.jdbc` (Laboratório no REPL)

**Objetivo:** aprender como o `next.jdbc` funciona **em isolamento**, antes de integrá-lo. Vamos criar um banco, inserir e ler dados, tudo a partir do REPL — sem tocar em nenhum arquivo do projeto.

**Ação 1: Inicie um REPL novo**

1. **Pare** todos os servidores.
2. Na raiz do projeto:

   ```bash
   clj
   ```

   (O `clj` lerá o `deps.edn` e baixará as novas bibliotecas na primeira vez.)

**Ação 2: Carregue a biblioteca e defina a conexão**

Digite no prompt `user=>`:

```clojure
user=> (require '[next.jdbc :as jdbc])
nil

;; A "especificação" do banco: um simples mapa Clojure
;; que diz ao next.jdbc como se conectar.
user=> (def db-spec {:dbtype "sqlite"    ;; o tipo de banco
                     :dbname "lab.db"})  ;; o arquivo onde tudo será salvo
#'user/db-spec
```

(Estamos usando `lab.db` como "banco de laboratório" descartável — o banco de verdade da aplicação, `prod.db`, será criado no próximo passo.)

**Ação 3: Execute SQL!** A principal função é `jdbc/execute!`:

```clojure
;; --- 1. CRIAR UMA TABELA (CREATE) ---
;; (execute! recebe um vetor cujo primeiro item é a string SQL)
user=> (jdbc/execute! db-spec ["
  CREATE TABLE IF NOT EXISTS todos (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT,
    completed BOOLEAN
  )
"])
[#:next.jdbc{:update-count 0}]

;; --- 2. INSERIR DADOS (INSERT) ---
;; (Passamos os valores como '?' para prevenir "SQL Injection".
;;  O next.jdbc cuida da substituição com segurança.)
user=> (jdbc/execute! db-spec
         ["INSERT INTO todos (title, completed) VALUES (?, ?)"
          "Aprender next.jdbc" false])
[#:next.jdbc{:update-count 1}]

user=> (jdbc/execute! db-spec
         ["INSERT INTO todos (title, completed) VALUES (?, ?)"
          "Integrar com a API" false])
[#:next.jdbc{:update-count 1}]

;; --- 3. LER OS DADOS (SELECT) ---
;; ESTE É O MOMENTO "AHA!":
;; o next.jdbc executa o SQL e retorna... DADOS CLOJURE!
;; (Um vetor de mapas — exatamente o que nossa API precisa.)
user=> (jdbc/execute! db-spec ["SELECT * FROM todos"])
[{:todos/id 1, :todos/title "Aprender next.jdbc", :todos/completed 0}
 {:todos/id 2, :todos/title "Integrar com a API", :todos/completed 0}]

;; --- 4. LER UM ITEM (SELECT com WHERE) ---
user=> (jdbc/execute! db-spec ["SELECT * FROM todos WHERE id = ?" 2])
[{:todos/id 2, :todos/title "Integrar com a API", :todos/completed 0}]
```

**Pare e observe duas coisas no resultado do SELECT:**

1. **As chaves vêm "qualificadas"**: não é `:id`, é **`:todos/id`** — o `next.jdbc` prefixa cada coluna com o nome da tabela. Guarde isso: **vai causar um bug no nosso frontend daqui a pouco** (e vamos corrigi-lo de propósito, no Passo 5.5).
2. **Booleanos viram números**: o SQLite armazena `true`/`false` como `1`/`0`. Isso também reaparecerá na Fase 6.

Saia do REPL com `Ctrl+D`. Olhe a pasta do projeto: há um novo arquivo `lab.db`. Nosso banco é real! Como foi só um laboratório, pode apagá-lo:

```bash
rm lab.db
```

- **A Lição: `next.jdbc` vs. o resto (o "porquê")**

  **1. `next.jdbc` vs. ORMs (Django, GORM, SQLAlchemy):**
  - **ORMs (mundo Objeto):** tentam **esconder** o SQL. Você lida com objetos e métodos (`user.save()`), e o ORM _adivinha_ o SQL. Fácil no começo, difícil de otimizar depois.
  - **`next.jdbc` (mundo Funcional/Dados):** **abraça** o SQL. Você escreve o SQL exato que quer, e a biblioteca é só uma **função**: `(função ["SQL..." dados]) → dados-clojure`. Transparente e focado em **transformação de dados**.

  **2. `next.jdbc` vs. Java JDBC (o "antes e depois"):**

  Para um simples `SELECT *` em Java JDBC puro, você escreveria toda esta "cerimônia":

  ```java
  Connection conn = null;
  PreparedStatement ps = null;
  ResultSet rs = null;
  List<Map<String, Object>> results = new ArrayList<>();

  try {
      conn = DriverManager.getConnection("jdbc:sqlite:prod.db");
      ps = conn.prepareStatement("SELECT * FROM todos");
      rs = ps.executeQuery();
      while (rs.next()) {
          Map<String, Object> row = new HashMap<>();
          row.put("id", rs.getInt("id"));
          row.put("title", rs.getString("title"));
          results.add(row);
      }
  } catch (SQLException e) {
      e.printStackTrace();
  } finally {
      if (rs != null) { rs.close(); }
      if (ps != null) { ps.close(); }
      if (conn != null) { conn.close(); }
  }
  return results;
  ```

  Com `next.jdbc`:

  ```clojure
  (jdbc/execute! db-spec ["SELECT * FROM todos"])
  ;;=> [{:todos/id 1, :todos/title "Aprender next.jdbc", ...}]
  ```

  O `next.jdbc` cuida de **toda** a cerimônia: abrir a conexão, criar o _statement_, executar, iterar o `ResultSet`, construir os mapas e fechar tudo com segurança.

### Passo 5.3: O Novo `db.clj` (com SQL)

Agora vamos usar o que aprendemos para **substituir permanentemente** o `db.clj` antigo. O objetivo: trocar o "motor" do carro sem que o motorista (o `handler.clj`) perceba.

**Ação:** **Apague todo o conteúdo** de `src/todo/backend/db.clj` e substitua por:

```clojure
(ns todo.backend.db
  "Este namespace gerencia os dados dos 'todos'
   usando um banco de dados persistente SQLite."
  (:require [next.jdbc :as jdbc]))

;; --- 1. A Configuração (Especificação do Banco) ---
;; O banco será salvo em um arquivo chamado "prod.db",
;; criado automaticamente na raiz do projeto.
(def db-spec {:dbtype "sqlite"
              :dbname "prod.db"})

;; --- 2. Função de Inicialização ---
;; Cria a tabela "todos" se ela ainda não existir.
;; Vamos chamá-la no 'core.clj' quando o servidor iniciar.
(defn initialize-database!
  "Cria a tabela 'todos' no banco de dados se ela não existir."
  []
  (jdbc/execute! db-spec ["
    CREATE TABLE IF NOT EXISTS todos (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT,
      description TEXT,
      completed BOOLEAN DEFAULT 0,
      created_at TEXT
    )
  "]))

;; --- 3. As Novas Funções (substituindo os 'atoms') ---

(defn get-all-todos
  "Retorna uma lista com todos os 'todos' no banco."
  []
  ;; O 'execute!' roda o SQL e já retorna mapas Clojure!
  (jdbc/execute! db-spec ["SELECT * FROM todos ORDER BY created_at DESC"]))

(defn create-todo
  "Cria um novo 'todo', salva no banco e o retorna."
  [todo-data] ;; ex: {:title "Minha tarefa", :description "..."}
  ;; A cláusula 'RETURNING *' pede ao SQLite para devolver
  ;; a linha recém-inserida (com o id gerado, o completed, etc.).
  ;; 'execute-one!' retorna esse único resultado como um mapa.
  (jdbc/execute-one! db-spec ["
    INSERT INTO todos (title, description, completed, created_at)
    VALUES (?, ?, ?, ?)
    RETURNING *"
    (:title todo-data)
    (:description todo-data)
    0 ;; completed = falso (lembre-se: SQLite usa 0/1)
    (str (java.time.Instant/now))]))
```

### O que fizemos?

1. **Removemos os `atom`s**: `todos-db` e `next-id` desapareceram.
2. **`(:require [next.jdbc :as jdbc])` dentro do `ns`**: exatamente o padrão que aprendemos na Fase 1 — os `require`s de um arquivo sempre vivem na declaração do namespace.
3. **`initialize-database!`**: o `CREATE TABLE IF NOT EXISTS` configura o banco na primeira execução e não faz nada nas seguintes.
4. **`get-all-todos`**: substituímos `(vals @todos-db)` por um `SELECT`.
5. **`create-todo`**: substituímos o `swap!` por um `INSERT ... RETURNING *`.

!!! tip
    **O que é o `RETURNING *`?** É uma cláusula SQL (suportada pelo SQLite desde a versão 3.35) que faz o comando `INSERT`/`UPDATE`/`DELETE` **devolver as linhas afetadas**, como se fosse um `SELECT`. Sem ela, um `INSERT` retornaria apenas `{:next.jdbc/update-count 1}` ("1 linha afetada") — e nosso handler responderia o objeto errado ao cliente. Com ela, `execute-one!` nos entrega o todo completo, com o `id` que o banco acabou de gerar. Usaremos o mesmo truque no `UPDATE` e no `DELETE` da Fase 6.


**A "Mágica":** os _nomes_ das funções (`get-all-todos`, `create-todo`) e o formato do que elas retornam (lista de mapas / mapa) são **os mesmos** de antes. Por isso o `handler.clj` **não precisa de nenhuma modificação** — ele nem vai "saber" que trocamos um `atom` por SQL.

### Passo 5.4: A Modificação no `core.clj`

Nosso novo `db.clj` tem a função `initialize-database!`, mas ela nunca é chamada. Precisamos executá-la **uma vez**, quando o servidor de backend iniciar.

**Ação 1: O `require`**

No topo do `src/todo/backend/core.clj`, adicione o `todo.backend.db` à lista de `require`s (atenção aos parênteses — o `(:gen-class)` continua _dentro_ do `ns`, como sempre esteve):

```clojure
(ns todo.backend.core
  (:require [ring.adapter.jetty :as jetty]
            [reitit.ring :as ring]
            [ring.middleware.json :refer [wrap-json-response wrap-json-body]]
            [ring.middleware.params :refer [wrap-params]]
            [ring.middleware.keyword-params :refer [wrap-keyword-params]]
            [ring.middleware.cors :refer [wrap-cors]]
            [todo.backend.handler :as handler]
            ;; --- ADICIONE ESTA LINHA ---
            [todo.backend.db :as db])
  (:gen-class))
```

**Ação 2: Chamar a inicialização no `-main`**

Encontre a função `-main` no final do arquivo. _Mude esta versão:_

```clojure
(defn -main [& args]
  (let [port (Integer/parseInt (or (first args) "3000"))]
    (start-server port)))
```

_Para esta:_

```clojure
(defn -main [& args]
  (let [port (Integer/parseInt (or (first args) "3000"))]
    ;; --- ADICIONE ESTA LINHA ---
    (db/initialize-database!) ;; Garante que a tabela exista
    ;; ---------------------------
    (start-server port)))
```

Na primeira execução, isso criará o `prod.db` e a tabela `todos`. Nas seguintes, o `CREATE TABLE IF NOT EXISTS` simplesmente não fará nada — exatamente o que queremos.

### Passo 5.5: A Correção de Acoplamento (Frontend)

**Objetivo:** antes de testar, precisamos corrigir uma incompatibilidade de "dialeto" entre o backend e o frontend — aquela que você viu nascer no laboratório do REPL.

**A Lição (o "porquê"):**

1. **Nosso backend (`next.jdbc`)** agora retorna _keywords qualificadas_ ("namespaced"): `:todos/id`, `:todos/title`, `:todos/completed`.
2. **Nosso frontend** espera _keywords simples_: `(:id todo)` e `(:title todo)`.

Se não corrigirmos, o frontend tentará ler `(:id todo)` de um mapa `{:todos/id 1, ...}`, receberá `nil`, e a UI quebrará: a lista mostrará _bullet points_ sem texto, e o console exibirá avisos de `:key`.

**A solução (deste tutorial):** corrigir o **frontend** para entender o "dialeto" do backend.

**A consequência (o "acoplamento"):** é a solução mais rápida, mas cria **acoplamento**: o frontend agora "sabe" que os dados vêm de uma tabela chamada `todos`. Se um dia a tabela for renomeada para `tasks`, o backend continuaria funcionando, mas o frontend quebraria. _(A solução "desacoplada" seria configurar o `next.jdbc` no backend para retornar keywords simples — fica como exercício para os curiosos: procure por `next.jdbc.result-set/as-unqualified-maps`.)_

**Ação:** Abra `src/todo/frontend/core.cljs` e encontre o componente `todo-list`.

!!! warning
    **Atenção: são DOIS lugares para corrigir, não um!** O erro mais comum deste passo é corrigir o `(:title ...)` (porque o texto sumido é visível) e **esquecer o `^{:key ...}`** (porque o aviso fica escondido no console). Se a key ficar errada, o app até "funciona", mas o React perde a identidade dos itens da lista — e isso causará bugs visuais reais na Fase 6 (checkboxes marcando o item errado!). Corrija os dois.


_Mude esta versão:_

```clojure
(defn todo-list []
  [:ul.todo-list
   (for [todo (:todos @app-state)]
     ^{:key (:id todo)}        ;; <-- Bug 1 aqui
     [:li.todo-item
      (:title todo)])])        ;; <-- Bug 2 aqui
```

_Para esta versão (corrigida nos dois lugares):_

```clojure
(defn todo-list []
  [:ul.todo-list
   (for [todo (:todos @app-state)]
     ^{:key (:todos/id todo)}  ;; <-- CORRIGIDO (1/2)
     [:li.todo-item
      (:todos/title todo)])])  ;; <-- CORRIGIDO (2/2)
```

Salve o arquivo.

### Passo 5.6: O Teste Final (A Persistência Real)

**Ação 1: Limpe qualquer banco antigo (recomendado)**

Para garantir que estamos começando do zero:

```bash
rm -f prod.db
```

**Ação 2: Inicie os dois servidores**

1. **Terminal 1 (Backend):**

   ```bash
   clj -M:run
   ```

   (O `prod.db` e a tabela `todos` serão criados. `Servidor iniciado na porta 3000`.)

2. **Terminal 2 (Frontend):**

   ```bash
   npx shadow-cljs watch app
   ```

**Ação 3: Teste a aplicação**

1. Abra `http://localhost:8000`.
2. A lista deve estar vazia — **e sem erros no console (F12)**.
3. Adicione um todo (ex: "Testar o banco SQLite").
4. Ele deve aparecer na lista, **com o texto visível**.

**Ação 4: O "Aha!" Final (o teste de persistência)**

Agora, o teste que **falhava** no fim da Fase 4:

1. Vá ao **Terminal 1 (Backend)** e **pare o servidor** (`Ctrl+C`).
2. **Reinicie-o**: `clj -M:run`.
3. No navegador, **recarregue a página (F5)**.

**Resultado Esperado:** o todo **continua lá!** O backend reiniciado leu o arquivo `prod.db` do disco e devolveu os dados. Persistência real. 🎉

### Passo 5.7: Git Checkpoint

**Ação 1:** pare os servidores e verifique:

```bash
git status
```

Você deve ver modificados: `deps.edn`, `src/todo/backend/core.clj`, `src/todo/backend/db.clj` e `src/todo/frontend/core.cljs`.

Note que o `prod.db` **não aparece** na lista — o `.gitignore` que escrevemos lá na Fase 0 (regra `*.db`) já está fazendo o trabalho dele. O arquivo de banco é _resultado_ da aplicação, não código-fonte, e jamais deve ir para o GitHub.

**Ação 2:** prepare e salve:

```bash
git add .
git commit -m "refactor(db): substitui banco em memória por persistência SQLite"
```

(Usamos `refactor:` porque mudamos a _implementação_ do banco sem mudar o comportamento externo da API.)

---

**Fim da Fase 5!** 🏁

Você tem uma aplicação full-stack completa, moderna e **persistente**. Mas nosso Todo List ainda só _cria_ e _lista_. Na próxima (e última) fase de código, completaremos o CRUD: marcar como feito (**U**pdate) e deletar (**D**elete) — com um visual decente.

---

