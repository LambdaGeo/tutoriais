# Elixir e Phoenix LiveView: Construindo uma Aplicação Todo List do Zero

*Um guia prático e completo do ecossistema Phoenix, explorando schemas, migrations, changesets e a reatividade em tempo real com LiveView.*

Este é um tutorial completo e guiado para construir uma aplicação de **Lista de Tarefas (Todo List)** do zero, usando a stack moderna de **Elixir com Phoenix LiveView** — um framework funcional e reativo para o desenvolvimento web full-stack.

Mas há um diferencial: este guia é o **segundo lado de uma mesma jornada**.

No outro tutorial — [Clojure/ClojureScript: Construindo uma Aplicação Persistente e Reativa](../clojure/index.md) — resolvemos o mesmo problema usando a stack **Clojure + Reagent + next.jdbc**, explorando o modelo de atualização reativa no navegador e a comunicação via API REST.

Aqui, faremos o mesmo **conceitualmente**, mas com **Elixir e LiveView**, onde **frontend e backend se fundem** em um único processo funcional e altamente performático.

---

### 🎯 Objetivo Pedagógico

O propósito deste tutorial não é apenas "fazer funcionar", mas **entender o porquê**. Cada comando, cada função e cada módulo será explicado com contexto e analogia. Você aprenderá:

- Como o Phoenix organiza um projeto web;
- O que são **schemas**, **migrations** e **changesets** (e como se relacionam com os _models_ dos ORMs tradicionais);
- Como o **LiveView** elimina a separação rígida entre frontend e backend, permitindo **interações em tempo real** sem escrever uma linha de JavaScript;
- E, claro, como criar, listar, marcar e excluir tarefas com atualização instantânea na interface.

---

### ⚙️ A Stack que Vamos Usar

- **Linguagem:** Elixir (baseada em Erlang, funcional e concorrente);
- **Framework Web:** Phoenix 1.8 + LiveView 1.1;
- **Banco de Dados:** SQLite (via Ecto);
- **Estilo:** Tailwind CSS v4 + daisyUI (já integrados ao Phoenix).

### 📌 Versões Utilizadas (Importante!)

Para que o tutorial seja reprodutível, estas são as versões de referência:

| Ferramenta / Biblioteca | Versão    |
| ----------------------- | --------- |
| Erlang/OTP              | 26+       |
| Elixir                  | 1.17+     |
| Node.js                 | 18+ (LTS) |
| Phoenix (`phx_new`)     | 1.8.x     |
| Phoenix LiveView        | 1.1.x     |
| `ecto_sql`              | ~> 3.10   |
| `ecto_sqlite3`          | ~> 0.12   |

!!! warning
    **Atenção especial ao Phoenix 1.8.** Muitos tutoriais e respostas antigas na internet (e em IAs!) referem-se ao Phoenix **1.7**, que usava Tailwind v3 com `tailwind.config.js` e componentes ligeiramente diferentes. O Phoenix **1.8** mudou várias dessas coisas. Este tutorial está inteiramente alinhado ao 1.8 — se algo que você encontrar por aí divergir daqui, desconfie da versão.


---

### 🔁 Um Mesmo Problema, Dois Caminhos Funcionais

| Aspecto      | Clojure                                  | Elixir                                            |
| ------------ | ---------------------------------------- | ------------------------------------------------- |
| Paradigma    | Funcional puro (imutabilidade explícita) | Funcional concorrente (processos leves)           |
| Renderização | Frontend reativo com Reagent (React)     | LiveView (renderização no servidor em tempo real) |
| Comunicação  | API REST + JSON                          | Canal WebSocket interno (phx)                     |
| Persistência | next.jdbc + SQLite                       | Ecto + SQLite                                     |
| Reatividade  | `r/atom` (frontend)                      | Estado do socket (`assigns`)                      |

Ambos ensinam o mesmo princípio:

> **como o estado flui em uma aplicação funcional e reativa.**

---

### 🧾 Os Marcos do Git (Seu Histórico Final)

Este tutorial também é um exercício de **desenvolvimento incremental com Git**. Cada fase termina em um commit; ao final, seu `git log --oneline` deve contar esta história (do mais recente para o mais antigo):

```
Fase 8: Atualiza README com instruções de execução
Fase 7: Ajusta o tema e personaliza o visual (Tailwind/daisyUI)
Fase 6: Implementa conclusão de tarefas (toggle_complete)
Fase 5: Implementa exclusão de tarefas (delete)
Fase 4: Refatora TodoLive para usar Ecto, Repo e to_form()
Fase 3: Persistência - Configura Ecto, Repo, Migrations e Task Schema
Fase 2: Lógica em Memória - Implementa adição de tarefas
Fase 1: Prova de Vida - Substitui a rota raiz por TodoLive
Fase 0: Gera o esqueleto do Phoenix com LiveView (sem Ecto)
Fase 0: Inicializa o repositório e .gitignore
```

---

### 📚 Índice das Fases

- **Fase 0:** [⚙️ Setup (Ambiente, Git e o Esqueleto do Projeto)](00-setup.md)
- **Fase 1:** [🏃 "Hello World" — Prova de Vida](01-hello-world.md)
- **Fase 2:** [🧠 Lógica em Memória](02-logica-memoria.md)
- **Fase 3:** [🧱 Persistência — A Camada de Dados (Ecto, Repo, Migration e Schema)](03-persistencia.md)
- **Fase 4:** [🫀 O "Transplante" — Conectando o LiveView ao Banco](04-transplante-liveview.md)
- **Fase 5:** [🗑️ Refinamento — Excluindo Tarefas](05-excluindo-tarefas.md)
- **Fase 6:** [✅ Refinamento — Concluindo Tarefas (Toggle)](06-concluindo-tarefas.md)
- **Fase 7:** [🎨 Personalizando o Design (Tailwind CSS v4 e daisyUI)](07-personalizando-design.md)
- **Fase 8:** [📄 README e Entrega](08-readme-entrega.md)

---

### 💡 Um Tutorial a Duas Mãos

Assim como o tutorial em Clojure, este também é fruto de uma construção colaborativa — unindo clareza didática com profundidade técnica.

Prepare seu ambiente, abra o terminal e venha ver como **Elixir + LiveView** transforma o jeito de pensar aplicações web. 🚀

---

