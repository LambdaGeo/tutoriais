# Fase 1: O Backend Mínimo (Servidor "Hello World")

**Objetivo:** Fazer um servidor web subir, rodar na sua máquina e responder "Hello, World!" quando acessado por uma URL. Isso prova que nossa configuração base está correta.

### Passo 1.1: O `deps.edn` (A "Lista de Compras" do Backend)

**O que é o `deps.edn`?**
Pense neste arquivo como a "lista de compras" do seu projeto. Ele diz ao Clojure (`clj`) quais bibliotecas (dependências) ele precisa baixar da internet para o projeto funcionar.

Ele também define "apelidos" (_aliases_), que são atalhos para comandos que usamos com frequência, como "rodar o servidor".

**Ação:** Crie o arquivo `deps.edn` na raiz do projeto (`todo-app/`) e cole o seguinte conteúdo:

```clojure
{:paths ["src" "resources"] ;; 1. Onde nosso código-fonte vai ficar

 :deps ;; 2. Nossa "lista de compras" de bibliotecas
 {;; O próprio Clojure
  org.clojure/clojure {:mvn/version "1.11.1"}

  ;; --- Dependências do Backend (API REST) ---
  ;; O "motor" do servidor web (Jetty) e as bibliotecas base do Ring
  ring/ring-core          {:mvn/version "1.12.2"}
  ring/ring-jetty-adapter {:mvn/version "1.12.2"}

  ;; A biblioteca de roteamento (para definir as URLs)
  metosin/reitit-ring     {:mvn/version "0.7.0"}}

 :aliases ;; 3. Nossos "atalhos" de comando
 {;; O alias que usaremos para iniciar o servidor
  :run
  {:main-opts ["-m" "todo.backend.core"]}}}
```

### O que fizemos?

1. **`:paths`**: dissemos ao Clojure para procurar nosso código nas pastas `src` e `resources` (que ainda vamos criar).
2. **`:deps`**: pedimos _apenas_ as bibliotecas essenciais do backend:
   - `ring/ring-jetty-adapter`: o servidor web que vai "ouvir" na `localhost:3000`.
   - `metosin/reitit-ring`: o "roteador" que olha a URL (ex: `/api/hello`) e decide qual função Clojure deve ser chamada.
   - **Nota:** ainda _não_ adicionamos `shadow-cljs` ou `reagent`. Faremos isso só na Fase 3, para manter o backend limpo.
3. **`:aliases`**: criamos o atalho `:run`. Quando rodarmos `clj -M:run`, ele executará a função principal (`-main`) do namespace `todo.backend.core` (que vamos criar a seguir).

### Passo 1.2: O Handler Mínimo (`handler.clj`)

**O que é um "Handler"?**
No mundo do **Ring** (a biblioteca base da web em Clojure), um _handler_ é simplesmente uma **função** que segue um contrato:

1. Ela recebe **um** argumento: um mapa `request` (com todos os dados da requisição HTTP que chegou).
2. Ela retorna **um** valor: um mapa `response` (que descreve a resposta que queremos enviar de volta).

Nosso objetivo é criar a `hello-handler` mais simples possível.

**Ação 1: Criar os diretórios**

O `deps.edn` diz ao Clojure para procurar código na pasta `src/`. Em Clojure, os namespaces são mapeados diretamente para a estrutura de pastas: `todo.backend.handler` deve viver no arquivo `src/todo/backend/handler.clj`.

No seu terminal (dentro de `todo-app/`), execute:

```bash
mkdir -p src/todo/backend
```

- `mkdir` cria diretórios; a flag `-p` cria todos os "diretórios pais" necessários no caminho, sem dar erro.

!!! tip
    **O que é um Namespace (`ns`)?**

    Em Clojure, não "importamos arquivos", nós "requeremos namespaces". Um namespace é um nome para um grupo de códigos, diretamente ligado à estrutura de pastas e ao nome do arquivo:

    | **Caminho do Arquivo**         | **Declaração de Namespace (no topo do arquivo)** |
    | ------------------------------ | ------------------------------------------------ |
    | `src/todo/backend/db.clj`      | `(ns todo.backend.db ...)`                       |
    | `src/todo/backend/handler.clj` | `(ns todo.backend.handler ...)`                  |

    Quando, em outro arquivo, quisermos usar as funções do `db.clj`, vamos "requerer" o namespace `todo.backend.db`, geralmente com um apelido (_alias_):

    ```clojure
    (ns todo.backend.handler
      (:require [todo.backend.db :as db])) ;; "db" agora é o apelido
    ```

    ⚠️ **Atenção a um detalhe que pega muita gente:** se o namespace tem um hífen no nome (ex: `db-config`), o **arquivo** usa _underscore_ (`db_config.clj`). Hífen no `ns`, underscore no nome do arquivo.


**Ação 2: Criar o arquivo do handler**

Crie o arquivo `src/todo/backend/handler.clj` e cole o seguinte código:

```clojure
(ns todo.backend.handler
  "Este namespace define nossas 'funções de resposta' (Handlers).")

(defn hello-handler
  "Nosso primeiro handler. Ele apenas diz 'Olá, Mundo!'"

  [_request] ;; 1. O handler recebe a 'request' como argumento.
             ;;    Usamos '_' para sinalizar que, nesta função,
             ;;    vamos ignorar esse argumento.

  ;; 2. O handler retorna um mapa de 'response'.
  {:status 200            ;; :status 200 é o código HTTP para "OK"
   :body "Hello, World!"}) ;; :body é o conteúdo enviado ao navegador
```

### O que fizemos?

Criamos nossa primeira peça de lógica: uma função pura e simples que atende ao contrato do Ring — ignora a entrada e retorna um mapa de resposta com status `200` e o texto "Hello, World!".

No entanto, essa função não faz nada sozinha. Precisamos de duas coisas:

1. Um **Servidor** (Jetty) para "ouvir" na `localhost:3000`.
2. Um **Roteador** (Reitit) para dizer: "quando chegar um `GET` em `/api/hello`, execute a `hello-handler`".

### Passo 1.3: O Servidor e o Roteador (`core.clj`)

O `core.clj` é o "cérebro" que junta todas as peças:

1. **Inicia o servidor** (Jetty), que escuta na porta `3000`.
2. **Define o roteador** (Reitit), que mapeia URLs para handlers.
3. É o **ponto de entrada** que o comando `clj -M:run` (definido no `deps.edn`) executa.

**Ação:** Crie o arquivo `src/todo/backend/core.clj` (na mesma pasta do `handler.clj`) e cole:

```clojure
(ns todo.backend.core
  (:require [ring.adapter.jetty :as jetty]       ;; 1. O software do Servidor (Jetty)
            [reitit.ring :as ring]               ;; 2. O Roteador (Reitit)
            [todo.backend.handler :as handler])  ;; 3. Nossas funções (handler.clj)
  (:gen-class))

;; --- 1. Definição das Rotas ---
;; A URL "/api/hello", quando acessada com o método :get,
;; deve executar nossa função handler/hello-handler.
(def app-routes
  (ring/router
   [["/api/hello" {:get {:handler handler/hello-handler}}]]))

;; --- 2. Definição da Aplicação (App) ---
;; O 'app' final é a função Ring principal.
(def app
  (ring/ring-handler
   app-routes                     ;; Nossas rotas
   (ring/create-default-handler))) ;; Um handler padrão para 404 (Not Found)

;; --- 3. Função para Iniciar o Servidor ---
(defn start-server [port]
  (println (str "Servidor iniciado na porta " port))
  ;; #'app passa a "var" da nossa app para o Jetty (útil no desenvolvimento)
  ;; :join? false evita que o servidor bloqueie a thread principal.
  (jetty/run-jetty #'app {:port port :join? false}))

;; --- 4. Ponto de Entrada Principal (-main) ---
;; Esta é a função que o alias :run (do deps.edn) procura.
(defn -main [& args]
  ;; Permite passar a porta como argumento (ex: clj -M:run 8080)
  ;; ou usa "3000" como padrão.
  (let [port (Integer/parseInt (or (first args) "3000"))]
    (start-server port)))
```

### O que fizemos?

1. **`(:require ...)`**: importamos nossas "ferramentas": Jetty, Reitit e nosso próprio `handler.clj`.
2. **`(:gen-class)`**: prepara este namespace para ser compilado como uma classe Java. Não é estritamente obrigatório para rodar com `clj -M:run`, mas é a convenção para namespaces com `-main` e será necessário se um dia você quiser empacotar a aplicação em um `.jar` executável. Vamos mantê-lo como boa prática.
3. **`app-routes`**: nosso "mapa do site". Por enquanto, com uma única rota.
4. **`app`**: a aplicação Ring principal, que "entrega" nossas rotas ao Jetty.
5. **`-main`**: a função que o `deps.edn` chama; pega a porta (ou usa `3000`) e chama `start-server`.

Neste ponto, temos as três peças: `deps.edn` (1.1), `handler.clj` (1.2) e `core.clj` (1.3). Vamos ver a mágica acontecer.

### Passo 1.4: Teste (Navegador e Terminal)

**Ação 1: Inicie o servidor**

No terminal, na raiz do projeto (onde está o `deps.edn`), execute:

```bash
clj -M:run
```

**Resultado Esperado:** Na primeira vez, o Clojure vai **baixar todas as dependências** (pode demorar um pouco — várias linhas de download aparecerão). Em seguida:

```
Servidor iniciado na porta 3000
```

**Importante:** este terminal agora está "ocupado" rodando o servidor. Deixe-o rodando.

**Ação 2: Teste no navegador**

1. Abra o navegador.
2. Digite a URL exata da nossa rota: `http://localhost:3000/api/hello`
3. Pressione Enter.

**Resultado Esperado:** a página deve mostrar apenas o texto do `:body` do nosso handler:

```
Hello, World!
```

**Ação 3: Teste no terminal com `curl`**

Para o restante do tutorial, usaremos bastante o `curl`, pois ele nos permite testar _todos_ os métodos HTTP (`GET`, `POST`, `DELETE`, etc.).

1. Abra um **novo** terminal (deixe o servidor rodando no primeiro).
2. Execute:

```bash
curl http://localhost:3000/api/hello
```

**Resultado Esperado:** o `curl` imprime o `:body` diretamente no terminal:

```
Hello, World!
```

### Se algo deu errado…

| Sintoma                                 | Causa provável                                                                                                           |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `Connection refused`                    | O servidor não está rodando no Terminal 1 (ou caiu com erro).                                                            |
| `404 Not Found`                         | Erro de digitação na URL ou na rota do `core.clj` (`/api/hello`).                                                        |
| `Could not locate todo/backend/core...` | O caminho do arquivo não bate com o namespace (confira `src/todo/backend/core.clj`) ou você não está na raiz do projeto. |
| Erro de sintaxe ao iniciar              | Algum parêntese a mais/menos ao colar. Compare com o código acima com calma.                                             |

### Passo 1.5: Git Checkpoint ("Hello World")

**Por que fazemos isso?**
Se, na próxima fase, ao adicionar a lógica do banco, quebrarmos tudo acidentalmente, teremos um "ponto seguro" para o qual podemos voltar.

**Ação 1:** No terminal do servidor, pare-o (`Ctrl+C`). Agora, veja o que o Git enxerga:

```bash
git status
```

**Resultado Esperado:** o Git mostrará os arquivos novos ("Untracked files"): `deps.edn` e `src/`.

**Ação 2:** Prepare e salve:

```bash
git add .
git commit -m "feat: implementa servidor 'Hello World' com Jetty e Reitit"
```

**Resultado Esperado:**

```
[main 1a2b3c4] feat: implementa servidor 'Hello World' com Jetty e Reitit
 3 files changed, ...
 create mode 100644 deps.edn
 create mode 100644 src/todo/backend/core.clj
 create mode 100644 src/todo/backend/handler.clj
```

---

**Fim da Fase 1!** 🏁

Temos um projeto Git limpo, com um servidor web "Hello World" totalmente funcional e testado. Agora estamos prontos para construir a lógica de negócios real: a API, começando pelo "banco de dados em memória" (`atom`).

---

