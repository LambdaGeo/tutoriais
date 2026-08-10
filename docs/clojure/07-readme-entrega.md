# Fase 7: README e Entrega

**Objetivo:** Escrever um `README.md` claro — a "porta de entrada" de qualquer repositório — e fazer o checklist final antes da entrega.

**Por que fazemos isso?**
Um projeto sem README é um projeto que ninguém consegue rodar. No mundo real, o README é o primeiro (e às vezes o único) arquivo que um colega, recrutador ou avaliador vai ler. Na avaliação desta disciplina, ele vale nota — e o critério é simples: _um colega conseguiria clonar e rodar seu projeto lendo apenas o README?_

### Passo 7.1: Criar o `README.md`

**Ação:** Crie o arquivo `README.md` na **raiz** do projeto e adapte o modelo abaixo (troque o nome, o link do repositório e o que mais quiser personalizar):

````markdown
# Todo List — Clojure/ClojureScript

**Aluno(a):** Seu Nome Completo Aqui

**Tutorial original:** [Tutorial Clojure/ClojureScript: Construindo uma Aplicação
Persistente e Reativa](https://github.com/LambdaGeo/tutoriais)

## Descrição

Aplicação full-stack de lista de tarefas (Todo List), construída de forma
incremental para estudar a arquitetura de aplicações funcionais modernas:

- **Backend:** Clojure, com [Ring](https://github.com/ring-clojure/ring)
  (servidor Jetty) e [Reitit](https://github.com/metosin/reitit) (roteamento),
  expondo uma API REST com CRUD completo.
- **Banco de dados:** SQLite, acessado via
  [next.jdbc](https://github.com/seancorfield/next-jdbc), com persistência
  real em disco (`prod.db`).
- **Frontend:** ClojureScript com
  [Reagent](https://github.com/reagent-project/reagent) (React) e
  [Shadow-CLJS](https://github.com/thheller/shadow-cljs), consumindo a API
  via `fetch`.

## Pré-requisitos

| Ferramenta                                                        | Versão mínima |
| ----------------------------------------------------------------- | ------------- |
| Java (JDK)                                                        | 11+           |
| [Clojure CLI](https://clojure.org/guides/install_clojure) (`clj`) | 1.11+         |
| Node.js (`node` / `npm`)                                          | 18+           |

## Como Rodar

1. **Clone o repositório e instale as dependências do frontend:**

   ```bash
   git clone https://github.com/SEU-USUARIO/SEU-REPO.git
   cd SEU-REPO
   npm install
   ```

2. **Terminal 1 — Backend (API na porta 3000):**

   ```bash
   clj -M:run
   ```

   Na primeira execução, as dependências Clojure serão baixadas e o banco
   `prod.db` será criado automaticamente.

3. **Terminal 2 — Frontend (porta 8000):**

   ```bash
   npx shadow-cljs watch app
   ```

   Aguarde a mensagem `Build completed`.

4. **Abra o navegador em:** [http://localhost:8000](http://localhost:8000)

## Endpoints da API

| Método   | Rota                    | Descrição                            |
| -------- | ----------------------- | ------------------------------------ |
| `GET`    | `/api/todos`            | Lista todas as tarefas               |
| `POST`   | `/api/todos`            | Cria uma tarefa (`{"title": "..."}`) |
| `POST`   | `/api/todos/:id/toggle` | Alterna o status feito/não feito     |
| `DELETE` | `/api/todos/:id`        | Remove uma tarefa                    |
````

!!! tip
    O modelo acima usa blocos de código aninhados dentro de uma lista — se o seu editor "quebrar" a renderização, simplifique: o importante é que as informações (nome, link, descrição, pré-requisitos e os comandos dos dois terminais) estejam lá e corretas.


### Passo 7.2: O Teste do "Colega"

**Ação:** Faça de conta que você é outra pessoa. Siga o **seu próprio README**, literalmente, do zero:

1. Clone seu repositório em uma **pasta nova** (ex: `/tmp/teste-todo`).
2. Execute exatamente os comandos do README, na ordem.
3. A aplicação subiu? O CRUD funciona? Os dados persistem após reiniciar o backend?

Se algo falhou, é o README (ou o repositório) que precisa de ajuste — melhor descobrir agora do que o avaliador descobrir depois. Um problema clássico revelado por esse teste: esquecer o passo `npm install` (a pasta `node_modules/` não vai para o Git!).

### Passo 7.3: Commit Final

```bash
git add README.md
git commit -m "docs: adiciona README com instruções de execução"
```

### Passo 7.4: Checklist de Entrega

Antes de enviar, confira o histórico:

```bash
git log --oneline
```

Você deve ver (de cima para baixo, do mais novo para o mais antigo):

```
docs: adiciona README com instruções de execução
feat(crud): implementa funcionalidades de toggle e delete
refactor(db): substitui banco em memória por persistência SQLite
feat: conecta frontend com API do backend (CORS corrigido)
feat: implementa UI do frontend com estado local (sem API)
feat: implementa API REST de 'todos' com banco em memória
feat: implementa servidor 'Hello World' com Jetty e Reitit
feat: setup inicial do projeto com .gitignore
```

(Commits extras no meio não são problema — o que importa é que os **marcos** estejam lá e na ordem certa.)

**Checklist final:**

- [ ] O repositório no GitHub é **público**?
- [ ] O `git push` foi feito? (O que está só na sua máquina não conta!)
- [ ] O README tem: nome completo, link do tutorial, descrição e instruções de execução?
- [ ] `prod.db` e `node_modules/` **não** estão no repositório (confira na página do GitHub)?
- [ ] O CRUD completo funciona e os dados persistem após reiniciar o backend?
- [ ] **SIGAA:** link público do repositório na caixa de **Comentários** + o **ZIP** (GitHub → botão _Code_ → _Download ZIP_) no **Anexo**.

---

## 🚀 Tutorial Concluído!

**Parabéns!** 🥳 Você construiu, passo a passo, uma aplicação web full-stack, moderna e persistente, depurando e corrigindo erros do mundo real ao longo do caminho — CORS, formatos de dados, keywords qualificadas, versões de ferramentas.

Mais importante do que o Todo List em si é o **padrão arquitetural** que você agora domina: estado imutável em caixas (`atom`/`r/atom`), funções puras como handlers, dados fluindo como mapas do banco ao navegador, e um histórico Git que conta a história do projeto.

**Quer ir além?** Algumas ideias de extensão (opcionais):

- Um campo de **edição** do título (o "U" completo do CRUD);
- Filtros "Todas / Ativas / Concluídas" no frontend;
- Exibir o `:error` e o `:loading` do `app-state` na interface;
- Trocar o SQLite por PostgreSQL (só muda o `db-spec` e o driver!);
- Refazer o mesmo problema em [Elixir/Phoenix LiveView](../elixir/index.md), no tutorial complementar da série.
