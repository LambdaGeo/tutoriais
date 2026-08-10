# Clojure e ClojureScript: Construindo uma Aplicação Todo List do Zero

*Um guia prático e completo de arquitetura funcional e reativa, cobrindo desde o backend Ring/Reitit até o frontend Reagent com SQLite.*

Olá e bem-vindo(a) a este guia prático.

O objetivo aqui é construir juntos uma aplicação **Todo List completa**, indo de um repositório Git vazio até um **projeto full-stack funcional**, usando o ecossistema **Clojure** moderno.

Mais do que um simples tutorial de "copiar e colar", este guia foi pensado para **ensinar arquitetura** — passo a passo, com atenção ao raciocínio funcional e à depuração de problemas reais.

Não vamos apenas construir uma aplicação: vamos entender **por que ela funciona** e **por que ela quebra**, explorando erros típicos (como CORS, formatos de dados incompatíveis e sincronização de estado) e aprendendo a corrigi-los com clareza.

Usaremos o clássico aplicativo **Todo List** como exemplo, pois sua simplicidade nos permite concentrar no que realmente importa: **a arquitetura e a interação entre as partes de um sistema reativo**.

---

### 🧱 O que Vamos Construir

- **Backend:** Clojure, com **Ring**, **Reitit** e **next.jdbc**;
- **Frontend:** ClojureScript, com **Reagent 2.0 (React 19)** e **Shadow-CLJS**;
- **Banco de Dados:** **SQLite**, para persistência real.

Seguiremos uma jornada incremental:

1. **Fundação:** verificação do ambiente, Git e `.gitignore`.
2. **Backend mínimo:** um servidor "Hello World".
3. **Banco em memória:** criando e lendo tarefas com um `atom`.
4. **Frontend reativo isolado:** interface dinâmica com Reagent.
5. **Integração full-stack:** comunicação via API REST, lidando com CORS e formato de dados.
6. **Banco real:** migrando para SQLite com `next.jdbc`.
7. **CRUD completo:** marcar tarefas como concluídas (**Update**) e removê-las (**Delete**), com um visual melhorado.
8. **Documentação:** um `README.md` profissional e a preparação para a entrega.

Ao final, você compreenderá **como os componentes de um sistema Clojure moderno se encaixam**, dominando o fluxo entre estado, renderização e persistência.

---

### 🗺️ A Arquitetura (Mapa Mental)

Durante quase todo o tutorial, você trabalhará com **dois terminais abertos ao mesmo tempo**:

| Terminal                  | Comando                     | O que roda                                      | Porta  |
| ------------------------- | --------------------------- | ----------------------------------------------- | ------ |
| **Terminal 1 (Backend)**  | `clj -M:run`                | A API REST (Jetty + Reitit)                     | `3000` |
| **Terminal 2 (Frontend)** | `npx shadow-cljs watch app` | O compilador CLJS + servidor de desenvolvimento | `8000` |

O navegador acessa sempre o **frontend** (`http://localhost:8000`), e o frontend conversa com o **backend** (`http://localhost:3000/api/...`) via `fetch`.

!!! tip
    Guarde esta tabela. A confusão mais comum do tutorial é acessar a porta errada ou esquecer que um dos dois servidores precisa estar rodando (ou precisa ser **reiniciado** após mudar o `deps.edn`).


---

### 📌 Versões Utilizadas (Importante!)

Para garantir que o tutorial seja reprodutível, **fixamos todas as versões**. Use exatamente estas — misturar versões (principalmente do `shadow-cljs`) é a causa nº 1 de erros difíceis de diagnosticar.

| Ferramenta / Biblioteca              | Versão                                    |
| ------------------------------------ | ----------------------------------------- |
| Java (JDK)                           | 11 ou superior (17 ou 21 recomendado)     |
| Clojure CLI (`clj` / `clojure`)      | 1.11+                                     |
| Node.js                              | 18 ou superior                            |
| `shadow-cljs` (npm **e** `deps.edn`) | `2.28.23` — **a mesma nos dois lugares!** |
| `reagent/reagent`                    | `2.0.0`                                   |
| `react` / `react-dom` (npm)          | `19.2.0`                                  |
| `ring` / `ring-jetty-adapter`        | `1.12.2`                                  |
| `metosin/reitit-ring`                | `0.7.0`                                   |
| `ring/ring-json`                     | `0.5.1`                                   |
| `ring-cors/ring-cors`                | `0.1.13`                                  |
| `seancorfield/next.jdbc`             | `1.2.659`                                 |
| `org.xerial/sqlite-jdbc`             | `3.45.3.0`                                |

> [!NOTE]
> Em sistemas operacionais onde a ferramenta `rlwrap` não estiver instalada, o comando de atalho `clj` pode falhar com o erro: `Please install rlwrap for command editing or use "clojure" instead.`
> Se isso acontecer no seu ambiente, basta substituir todas as chamadas a `clj` do tutorial pela palavra `clojure` (ex: use `clojure -M:run` em vez de `clj -M:run`, e `clojure` para abrir o REPL). Ambos executam exatamente a mesma engine Clojure CLI, mas o comando `clojure` funciona sem depender do editor de linha de comando `rlwrap`.

---

### 🧾 Os Marcos do Git (Seu Histórico Final)

Este tutorial é também um exercício de **desenvolvimento incremental com Git**. Ao final, seu `git log --oneline` deve contar esta história (do mais recente para o mais antigo):

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

Cada fase termina com um **Git Checkpoint** que gera exatamente um desses commits.

---

### 📚 Índice das Fases

- **Fase 0:** [A Fundação (Ambiente, Setup e Git)](00-fundacao.md)
- **Fase 1:** [O Backend Mínimo (Servidor "Hello World")](01-backend-minimo.md)
- **Fase 2:** [O Backend Funcional (API com Banco em Memória)](02-backend-funcional.md)
- **Fase 3:** [Introdução ao Frontend (Reagent e Shadow-CLJS)](03-frontend-intro.md)
- **Fase 4:** [Conectando o Frontend ao Backend (CORS e `fetch`)](04-integracao-frontend-backend.md)
- **Fase 5:** [Persistência Real (SQLite com `next.jdbc`)](05-persistencia.md)
- **Fase 6:** [🏆 CRUD Completo — "Marcar como Feito", "Deletar" e o Visual Final](06-crud-completo.md)
- **Fase 7:** [README e Entrega](07-readme-entrega.md)

Boa jornada! 🚀

---

