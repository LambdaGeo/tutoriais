# Fase 3: Introdução ao Frontend (Reagent e Shadow-CLJS)

**Objetivo:** Aprender os fundamentos do ClojureScript e do Reagent. Vamos construir uma UI interativa que gerencia seu _próprio_ estado (na memória do navegador), **sem** se comunicar com a API ainda.

!!! tip
    **Nesta fase, o servidor de backend (porta 3000) não é necessário.** Pode deixá-lo desligado. Vamos trabalhar apenas com o frontend (porta 8000).


### Passo 3.1: Setup do Ambiente Node.js (`npm`)

O `shadow-cljs` (nosso compilador de frontend) é uma ferramenta "híbrida": ele é uma biblioteca Clojure (que entrará no `deps.edn`), mas também é uma ferramenta de linha de comando que roda em **Node.js**. Além disso, o React será instalado via `npm`.

**Ação:** Na raiz do projeto (`todo-app/`), execute estes dois comandos:

```bash
# 1. Cria o package.json (o "deps.edn" do mundo Node.js)
npm init -y

# 2. Instala as ferramentas, com versões FIXADAS
npm install shadow-cljs@2.28.23 react@19.2.0 react-dom@19.2.0
```

**O que esses comandos fazem?**

1. **`npm init -y`** cria o arquivo `package.json`, que rastreia as dependências de JavaScript do projeto (o `-y` aceita todas as respostas padrão).
2. **`npm install ...`** baixa as ferramentas para a pasta `node_modules/` (que nosso `.gitignore` da Fase 0 já está, corretamente, ignorando) e as registra no `package.json`.

!!! warning
    **Por que fixamos as versões?** O `shadow-cljs` existe em **dois lugares**: como pacote `npm` (a linha de comando) e como biblioteca Maven no `deps.edn` (o compilador em si). **As duas versões precisam ser idênticas.** Se você rodar apenas `npm install shadow-cljs` (sem versão), o npm instalará a versão mais recente (3.x), que não vai bater com a que colocaremos no `deps.edn` — e você verá avisos de *"version mismatch"* e erros estranhos de compilação. Fixar `2.28.23` nos dois lugares elimina esse problema pela raiz.


### Passo 3.2: Configurar as Bibliotecas do Frontend (`deps.edn`)

**O que é o Reagent?**
O Reagent é uma biblioteca que nos permite usar o **React** (a popular biblioteca de UI) de uma forma muito limpa e "Clojure-nativa". Escrevemos vetores simples (chamados **Hiccup**), e o Reagent os transforma em componentes React super rápidos.

**Ação:** Abra o `deps.edn` e adicione as duas linhas do frontend ao bloco `:deps`:

```clojure
{:paths ["src" "resources"]

 :deps
 {org.clojure/clojure {:mvn/version "1.11.1"}

  ;; --- Dependências do Backend (API REST) ---
  ring/ring-core          {:mvn/version "1.12.2"}
  ring/ring-jetty-adapter {:mvn/version "1.12.2"}
  ring/ring-json          {:mvn/version "0.5.1"}
  metosin/reitit-ring     {:mvn/version "0.7.0"}

  ;; --- Dependências do Frontend ---
  ;; ADICIONE ESTAS DUAS LINHAS:
  ;; (a versão do shadow-cljs é a MESMA que instalamos via npm!)
  thheller/shadow-cljs    {:mvn/version "2.28.23"}
  reagent/reagent         {:mvn/version "2.0.0"}}

 :aliases
 {:run
  {:main-opts ["-m" "todo.backend.core"]}}}
```

### O que fizemos?

Nossa "lista de compras" Clojure agora inclui:

1. **`thheller/shadow-cljs`** `2.28.23`: o compilador — na mesma versão do pacote npm.
2. **`reagent/reagent`** `2.0.0`: a biblioteca de UI, na versão moderna que funciona com React 18/19.

### Passo 3.3: Configurar o Compilador (`shadow-cljs.edn`)

Temos as bibliotecas, mas precisamos de um arquivo para dizer ao `shadow-cljs` _como_ compilar nosso frontend. Este arquivo é o "cérebro" do shadow-cljs.

**Ação:** Crie um arquivo **novo** chamado `shadow-cljs.edn` na **raiz** do projeto (ao lado do `deps.edn`) e cole:

```clojure
{:deps true ;; 1. Informa ao shadow-cljs para usar o deps.edn

 :builds
 {:app ;; Damos ao nosso "build" (processo) o nome de ":app"
  {;; :browser = o destino do nosso código é um navegador web
   :target :browser

   ;; Onde o JavaScript compilado deve ser salvo
   :output-dir "resources/public/js"

   ;; O caminho que o navegador usará para acessar esses arquivos
   :asset-path "/js"

   ;; O "ponto de entrada" do nosso código.
   ;; Diz ao shadow-cljs para compilar o namespace 'todo.frontend.core'
   ;; e, quando a página carregar, chamar a função 'init'.
   :modules {:main {:init-fn todo.frontend.core/init}}

   ;; Configurações do servidor de desenvolvimento
   :devtools
   {;; O servidor serve os arquivos estáticos (como o index.html)
    ;; desta pasta:
    :http-root "resources/public"

    ;; O servidor roda na porta 8000 (diferente da API, na 3000)
    :http-port 8000}}}}
```

### O que fizemos?

1. **`:deps true`**: o shadow-cljs vai usar as dependências do nosso `deps.edn` (por isso as versões precisam bater!).
2. **`:output-dir`**: o JavaScript final irá para `resources/public/js` (pasta que o `.gitignore` já ignora).
3. **`:modules`**: o "ponto de partida" é a função `init` no namespace `todo.frontend.core` (que ainda não existe — criaremos já já).
4. **`:devtools`**: o servidor de desenvolvimento roda na **porta 8000** e serve os arquivos de `resources/public`.

### Passo 3.4: Criar o HTML Base

O shadow-cljs precisa de um `index.html` para servir ao navegador. Este arquivo é a "concha" onde nossa aplicação ClojureScript será carregada.

**Ação 1:** Crie a pasta que o servidor de desenvolvimento usa:

```bash
mkdir -p resources/public
```

**Ação 2:** Crie o arquivo `resources/public/index.html` e cole:

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Todo App (CLJS)</title>
  </head>
  <body>
    <div id="app">Carregando...</div>

    <script src="/js/main.js"></script>
  </body>
</html>
```

### O que fizemos?

1. **`<div id="app">`**: o "ponto de montagem" **crucial**. Quando nossa aplicação iniciar, ela vai procurar este `div` e injetar toda a interface dentro dele.
2. **`<script src="/js/main.js">`**: carrega o JavaScript compilado que o shadow-cljs vai gerar (o nome `main.js` vem do módulo `:main` do `shadow-cljs.edn`).

### Passo 3.5: "Olá, Reagent!" (UI Estática)

**Objetivo:** provar que o compilador consegue ler nosso arquivo `.cljs`, transformá-lo em JavaScript e injetá-lo no `index.html`. Sem estado, sem botões — apenas um "Olá".

**Ação 1:** Crie a pasta do frontend:

```bash
mkdir -p src/todo/frontend
```

**Ação 2:** Crie o arquivo `src/todo/frontend/core.cljs` e cole:

```clojure
(ns todo.frontend.core
  ;; Requeremos o "coração" do Reagent (para 'r/atom' e 'r/as-element')
  (:require [reagent.core :as r]
            ;; E o DOM do Reagent (para 'create-root', API do React 18+)
            [reagent.dom.client :as rdom]))

;; --- 1. O Componente ---
;; Um componente em Reagent é apenas uma função
;; que retorna "Hiccup" (HTML escrito como vetores CLJS).
;;
;; [:h1 "Olá"] -> <h1>Olá</h1>
(defn hello-world []
  [:div
   [:h1 "Olá, Alunos!"]
   [:p "Nossa aplicação ClojureScript está funcionando."]])

;; --- 2. A Inicialização (React 18+) ---
;; Esta é a função que o shadow-cljs.edn chama.
;; Ela "monta" nosso componente [hello-world]
;; no <div id="app"> do nosso index.html.
(defn ^:export init []
  (println "Frontend inicializado...")
  (let [root (rdom/create-root (js/document.getElementById "app"))]
    (.render root (r/as-element [hello-world]))))
```

**Ação 3: Compile e rode**

No terminal (pode ser o mesmo de antes, já que o backend está desligado):

```bash
npx shadow-cljs watch app
```

- Na **primeira vez**, o shadow-cljs vai baixar várias dependências — pode demorar alguns minutos. Espere até ver algo como `Build completed`.
- O comando `watch` fica **observando** seus arquivos: cada vez que você salvar um `.cljs`, ele recompila automaticamente (_hot-reload_). Deixe este terminal rodando — ele será o nosso **Terminal 2 (Frontend)** daqui em diante.

**Ação 4: Abra o navegador em** `http://localhost:8000`

**Resultado Esperado:** a página não deve mais mostrar "Carregando...". Em vez disso:

> **Olá, Alunos!**
> Nossa aplicação ClojureScript está funcionando.

### Se algo deu errado…

| Sintoma                                     | Causa provável                                                                                                                               |
| ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `shadow-cljs - dependency version mismatch` | A versão do shadow-cljs no `npm` é diferente da do `deps.edn`. Confira o Passo 3.1/3.2: ambas devem ser `2.28.23`.                           |
| Página continua em "Carregando..."          | O build falhou. Olhe o terminal do `watch`: haverá um erro de compilação apontando arquivo e linha. Console do navegador (F12) também ajuda. |
| `port 8000 already in use`                  | Outro processo está usando a porta. Pare-o ou mude o `:http-port` no `shadow-cljs.edn`.                                                      |
| `Cannot find module 'react'`                | O `npm install` do Passo 3.1 não foi executado (ou foi executado em outra pasta).                                                            |

### Passo 3.6: O "Cérebro" Reativo (o `r/atom`)

**A ligação: `atom` (Backend) vs. `r/atom` (Frontend)**

Na **Fase 2**, aprendemos o `atom` do Clojure:

- **O problema:** a imutabilidade — não podíamos "mudar" um mapa.
- **A solução:** uma "caixa" segura onde _trocamos_ (`swap!`) o valor antigo por um novo.

No **frontend**, temos um problema parecido: como a UI (o HTML) vai "saber" quando um valor muda? A solução do Reagent é a sua própria versão do atom: `reagent.core/atom` (que apelidamos de `r/atom`).

**Qual é a diferença?**

1. **Sintaxe:** nenhuma! Você usa `@` para ler, `swap!` para atualizar e `reset!` para substituir — exatamente como aprendeu.
2. **A "mágica":** o `r/atom` é uma caixa que **"toca um sino"**. Qualquer componente que "lê" um `r/atom` (usando `@`) está "ouvindo" esse sino. Quando você usa `swap!` ou `reset!`, o sino toca, e o Reagent redesenha automaticamente _apenas_ os componentes que estavam ouvindo.

Vamos provar isso com um contador.

**Ação 1:** _Substitua_ todo o conteúdo do `src/todo/frontend/core.cljs` por:

```clojure
(ns todo.frontend.core
  (:require [reagent.core :as r]
            [reagent.dom.client :as rdom]))

;; --- 1. O "Cérebro" (o r/atom) ---
;;
;; (defonce) garante que o valor (o 0) NÃO seja resetado
;; toda vez que o "hot-reload" do shadow-cljs recompilar.
(defonce app-state (r/atom 0))

;; --- 2. O Componente ---
(defn counter-app []
  [:div
   [:h1 "Entendendo o 'r/atom'"]

   ;; (Leitura): "lemos" o valor dentro da caixa usando '@'
   [:p "O valor atual do contador é: " @app-state]

   [:button
    ;; (Escrita): no clique, usamos 'swap!' para "trocar"
    ;; o valor na caixa, aplicando a função 'inc'.
    {:on-click #(swap! app-state inc)}
    "Clique para Incrementar"]

   [:button
    {:on-click #(reset! app-state 0)}
    "Resetar"]])

;; --- 3. A Inicialização ---
(defn ^:export init []
  (println "Frontend 'Contador' inicializado...")
  (let [root (rdom/create-root (js/document.getElementById "app"))]
    (.render root (r/as-element [counter-app]))))
```

**Ação 2 (Teste):**

1. Salve o arquivo. O shadow-cljs (Terminal 2) recompila sozinho.
2. Vá ao navegador em `http://localhost:8000`.
3. **Teste a mágica:** clique nos botões. O número muda instantaneamente, sem recarregar a página.

**A lição:** você acabou de aprender o "coração" do Reagent: `(defonce app-state (r/atom ...))` (o cérebro), `@app-state` (leitura) e `swap!`/`reset!` (escrita).

### Passo 3.7: A UI Estática do Todo (com dados falsos)

**Objetivo:** agora que sabemos _como_ o Reagent funciona, vamos construir o _visual_ (estático) do Todo App. Ele não vai funcionar ainda — apenas parecer correto.

**Ação:** _Substitua_ o conteúdo do `core.cljs` (o contador) por este, agora separado em componentes:

```clojure
(ns todo.frontend.core
  (:require [reagent.core :as r]
            [reagent.dom.client :as rdom]))

;; --- 1. Nossos Componentes (Estáticos) ---

;; O Formulário (não faz nada ainda)
(defn todo-form []
  [:div.todo-input
   [:input {:type "text" :placeholder "O que precisa ser feito?"}]
   [:button "Adicionar (Desligado)"]])

;; A Lista (recebe uma lista de "todos" como argumento)
(defn todo-list [todos] ;; "todos" é um argumento!
  [:ul.todo-list
   (for [todo todos]
     ^{:key (:id todo)}
     [:li.todo-item
      (:title todo)])])

;; O App Principal (que monta tudo)
(defn app []
  [:div.todo-app
   [:h1 "Todo App (Estático)"]
   [:p "Isto é apenas o visual. Nada funciona ainda."]

   ;; 1. Renderiza o formulário
   [todo-form]

   ;; 2. Renderiza a lista, passando "dados falsos"
   [todo-list [{:id 1 :title "Meu primeiro item (falso)"}
               {:id 2 :title "Meu segundo item (falso)"}]]])

;; --- 2. A Inicialização ---
(defn ^:export init []
  (println "Frontend 'Todo Estático' inicializado...")
  (let [root (rdom/create-root (js/document.getElementById "app"))]
    (.render root (r/as-element [app]))))
```

!!! tip
    **O que é o `^{:key ...}`?** Quando renderizamos uma lista com `for`, o React precisa de uma "identidade" para cada item, para saber o que mudou entre uma renderização e outra. O metadado `^{:key (:id todo)}` fornece essa identidade. Sem ele, o console mostra o aviso *"Every element in a seq should have a unique :key"*. **Guarde esse detalhe: ele vai reaparecer na Fase 5.**


**Teste:** salve e veja no navegador. Você deve ver a UI completa, com os dois todos falsos listados. O campo e o botão estão lá, mas clicar não faz nada.

### Passo 3.8: Conectando a UI ao `r/atom` (Tornando Interativo)

Agora vamos fazer o formulário funcionar de verdade — mas ainda **100% local**, sem API.

**Ação 1: Adicionar o `clojure.string`**

No `ns` do `core.cljs`, precisamos do `clojure.string` para checar se o input está em branco. _Mude isto:_

```clojure
(ns todo.frontend.core
  (:require [reagent.core :as r]
            [reagent.dom.client :as rdom]))
```

_Para isto:_

```clojure
(ns todo.frontend.core
  (:require [reagent.core :as r]
            [reagent.dom.client :as rdom]
            [clojure.string :as str])) ;; <- ADICIONE ESTA LINHA
```

**Ação 2: Adicionar o "Cérebro" (`app-state`) e a lógica local**

Logo abaixo do `ns`, **adicione** este bloco. Este é o `r/atom` central e a função local que o modifica:

```clojure
;; --- O "Cérebro" Reativo ---
(defonce app-state (r/atom {:next-id 1
                            :input-text ""
                            :todos []}))

;; --- A Lógica de Negócios (Local) ---
(defn adicionar-todo-local []
  (swap! app-state
         (fn [estado-atual]
           (let [novo-titulo (:input-text estado-atual)
                 novo-id (:next-id estado-atual)]
             (if (str/blank? novo-titulo)
               estado-atual ;; Não faz nada se vazio
               ;; Retorna um NOVO estado
               {:next-id (inc novo-id)
                :input-text "" ;; Limpa o input
                :todos (conj (:todos estado-atual)
                             {:id novo-id
                              :title novo-titulo})})))))
```

**Ação 3: Modificar o `todo-form`**

Encontre a função `todo-form` e conecte-a ao `app-state`. _Mude esta versão (estática):_

```clojure
(defn todo-form []
  [:div.todo-input
   [:input {:type "text" :placeholder "O que precisa ser feito?"}]
   [:button "Adicionar (Desligado)"]])
```

_Para esta versão (conectada):_

```clojure
(defn todo-form []
  [:div.todo-input
   [:input
    {:type "text"
     :placeholder "O que precisa ser feito?"
     ;; (Leitura): o valor do input vem do app-state
     :value (:input-text @app-state)
     ;; (Escrita): o on-change atualiza o app-state a cada tecla
     :on-change #(swap! app-state assoc :input-text (-> % .-target .-value))}]

   [:button
    ;; (Ação): o on-click chama nossa lógica local
    {:on-click adicionar-todo-local}
    "Adicionar (Local)"]])
```

**Ação 4: Modificar o `todo-list`**

Vamos remover o argumento `[todos]` e fazê-la ler diretamente do `app-state`. _Mude esta versão:_

```clojure
(defn todo-list [todos] ;; <-- Recebe "todos" como argumento
  [:ul.todo-list
   (for [todo todos]
     ^{:key (:id todo)}
     [:li.todo-item
      (:title todo)])])
```

_Para esta versão:_

```clojure
(defn todo-list [] ;; <-- Argumento "todos" REMOVIDO
  [:ul.todo-list
   ;; (Leitura): o 'for' agora observa o @app-state
   (for [todo (:todos @app-state)]
     ^{:key (:id todo)}
     [:li.todo-item
      (:title todo)])])
```

**Ação 5: Modificar o `app` (remover os dados falsos)**

_Mude esta versão:_

```clojure
(defn app []
  [:div.todo-app
   [:h1 "Todo App (Estático)"]
   [:p "Isto é apenas o visual. Nada funciona ainda."]

   [todo-form]

   [todo-list [{:id 1 :title "Meu primeiro item (falso)"}
               {:id 2 :title "Meu segundo item (falso)"}]]])
```

_Para esta versão (limpa e interativa):_

```clojure
(defn app []
  [:div.todo-app
   [:h1 "Todo App (Somente Frontend)"]
   [:p "Isto é 100% local. Recarregue (F5) para ver os dados sumirem."]

   ;; Os componentes agora se viram sozinhos!
   [todo-form]
   [todo-list]])
```

### Teste (A "Lição")

1. Salve o `core.cljs`. O shadow-cljs recompila.
2. Vá ao navegador em `http://localhost:8000`.

**Teste a interatividade:**

1. A lista deve estar vazia.
2. Digite "Meu primeiro todo" e clique em "Adicionar (Local)".
3. **Mágica:** o todo aparece na lista _instantaneamente_, e o input é limpo.

**O "Aha!":** agora, **recarregue a página (F5)**.

_Todos os todos que você adicionou desapareceram._

Isso prova que o estado vivia apenas na memória do navegador. Agora temos uma **motivação** clara para a próxima fase: como fazer essa lista persistir? A resposta é a nossa API do backend.

### Passo 3.9: Git Checkpoint

**Por que fazemos isso?**
Acabamos de construir uma "mini-aplicação" de frontend completa: `npm`, `shadow-cljs`, `react` e a reatividade do Reagent. É o ponto de salvamento perfeito antes de misturar a complexidade da API.

**Ação 1:** Pare o `shadow-cljs` (`Ctrl+C` no Terminal 2) e verifique:

```bash
git status
```

Você deve ver os novos arquivos e modificações: `package.json`, `package-lock.json`, `shadow-cljs.edn`, `resources/public/index.html`, `src/todo/frontend/core.cljs` e o `deps.edn` modificado. (Repare que `node_modules/` e `resources/public/js/` **não aparecem** — obrigado, `.gitignore`!)

**Ação 2:** Prepare e salve — este é **um único commit**, com exatamente esta mensagem (ela é um dos marcos avaliados):

```bash
git add .
git commit -m "feat: implementa UI do frontend com estado local (sem API)"
```

---

**Fim da Fase 3!** 🏁

Temos um backend funcional (Fase 2) e um frontend funcional (Fase 3), em commits separados. Eles ainda não se "conhecem".

---

