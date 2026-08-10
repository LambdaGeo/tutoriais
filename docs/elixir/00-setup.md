# ⚙️ Fase 0: Setup (Ambiente, Git e o Esqueleto do Projeto)

**Objetivo:** Preparar o ambiente, instalar as ferramentas e gerar o esqueleto do projeto. Mas antes, vamos entender o que é essa stack.

### A Stack: Phoenix LiveView

Antes de começar, é crucial entender por que o **LiveView** é uma abordagem diferente — e por que ela vem ganhando tanto destaque entre desenvolvedores acostumados a React, Vue ou Next.js.

### O Modo Tradicional (React / Next.js)

Em um modelo tradicional baseado em frameworks JavaScript, a aplicação é dividida em **duas camadas independentes**:

1. **Backend (API)** — armazena e serve dados, geralmente em **JSON**.
2. **Frontend (SPA ou SSR)** — construído em JavaScript, responsável pela interface, pelo estado e pela renderização dos componentes.

No **React puro**, o navegador recebe um HTML básico e depois baixa e executa o JavaScript que monta toda a interface (renderização no cliente). O **Next.js** otimiza isso com **SSR/SSG**, renderizando o HTML inicial no servidor — mas, após o carregamento, o React ainda assume o controle no cliente, "reidratando" a página.

Apesar de eficiente, esse modelo **mantém o estado do aplicativo no navegador**, exigindo sincronização constante com o backend via AJAX ou GraphQL. Isso significa **duas fontes de verdade** (cliente e servidor), **gerenciamento de estado complexo** e **múltiplas camadas de código** para manter tudo sincronizado.

### O Modo LiveView (Estado no Servidor)

O **Phoenix LiveView** propõe algo radicalmente diferente:

👉 **toda a lógica de estado e renderização vive no servidor.**

O navegador abre uma **conexão WebSocket** — um "túnel" bidirecional e contínuo — com o servidor Phoenix. A partir daí, toda interação do usuário (como clicar em "Adicionar Tarefa") envia apenas uma **mensagem leve** para o servidor: `"o evento 'save_task' aconteceu"`.

O **servidor Elixir** processa o evento, atualiza o estado (a lista de tarefas) e re-renderiza o HTML **no próprio servidor**. Em seguida, calcula o que mudou (o _diff_) e envia apenas os fragmentos atualizados de volta. Um **JavaScript minúsculo**, incluído automaticamente pelo LiveView, faz o "remendo" na página — sem recarregar, sem sincronizar estados, sem React Hooks, sem Redux.

### A Vantagem

- O **estado vive em um só lugar** — no servidor.
- Você escreve **quase zero JavaScript**.
- A interface é **reativa em tempo real** por padrão.
- A performance surpreende: o Elixir lida com **milhares de conexões simultâneas** graças ao modelo de concorrência leve da **BEAM VM**.

O LiveView traz a **simplicidade do desenvolvimento tradicional (HTML + servidor)** com a **interatividade do front moderno (SPA)** — sem manter duas aplicações separadas.

---

### 🧰 Passo 0.1: Instalar as Ferramentas

Precisamos de três coisas:

1. **Node.js** — o Phoenix o usa para compilar assets (CSS/JS);
2. **Erlang** — a máquina virtual (BEAM) sobre a qual o Elixir roda;
3. **Elixir** — a linguagem de programação.

**1. Node.js:** baixe e instale a versão **LTS** no [site oficial](https://nodejs.org/).

**2. Elixir e Erlang (via script oficial):** a maneira mais confiável, com controle exato das versões.

- **Linux/macOS (Bash):**

  ```bash
  # Baixa e executa o script, fixando as versões
  curl -fsSO https://elixir-lang.org/install.sh
  sh install.sh elixir@1.17.2 otp@26.2.2 installs_dir=$HOME/.elixir-install/installs

  # Adicione ao seu PATH (ex: no ~/.bashrc ou ~/.zshrc)
  # export PATH=$HOME/.elixir-install/installs/otp/26.2.2/bin:$PATH
  # export PATH=$HOME/.elixir-install/installs/elixir/1.17.2-otp-26/bin:$PATH
  ```

- **Windows (PowerShell):**

  ```powershell
  # Baixa e executa o script
  curl.exe -fsSO https://elixir-lang.org/install.bat
  .\install.bat elixir@1.17.2 otp@26.2.2

  # Adicione os diretórios ao seu PATH de Ambiente de Usuário
  # (ex: %USERPROFILE%\.elixir-install\installs\otp\26.2.2\bin)
  # (ex: %USERPROFILE%\.elixir-install\installs\elixir\1.17.2-otp-26\bin)
  ```

**Verificação:** feche e reabra o terminal, e confirme que tudo respondeu com uma versão:

```bash
elixir --version   # Elixir 1.17.x (compiled with Erlang/OTP 26)
node -v            # v18 ou superior
git --version
```

_(Se `elixir` não for encontrado, o `PATH` não foi configurado permanentemente — revise o passo acima antes de prosseguir.)_

### 🧱 Passo 0.2: Instalar o Hex e o Gerador do Phoenix

O **Mix** é a ferramenta de build do Elixir — algo como o **npm** (Node.js), o **pip** (Python) ou o **Maven** (Java). Com ele, gerenciamos dependências, rodamos tarefas, executamos testes e criamos projetos. Ele já vem instalado junto com o Elixir.

**O que é o Hex?** O **Hex** é o **gerenciador de pacotes oficial do Elixir** — o papel que o npm faz para o JavaScript. Quando usarmos bibliotecas externas (como o Ecto), é o Hex quem as baixa. Instale-o com:

```bash
mix local.hex
```

**O que é o Phoenix?** O **Phoenix** é o principal **framework web** do ecossistema Elixir — comparável ao **Django** (Python), **Rails** (Ruby) ou **Express** (Node.js), mas construído para aproveitar ao máximo a **concorrência** e o **tempo real** da BEAM VM. Ele vem com o **LiveView**, que permite construir interfaces reativas **sem JavaScript manual**.

Instale o gerador de projetos:

```bash
mix archive.install hex phx_new
```

✅ **Resumo:**

- **Mix** → ferramenta de build e tarefas (como `npm` ou `pip`);
- **Hex** → gerenciador de pacotes (como o registro do `npm`);
- **Phoenix** → framework web completo, com foco em performance e tempo real.

### 📁 Passo 0.3: O Diretório e o Git

O `Git` é nosso sistema de controle de versão. Vamos usá-lo desde o início para salvar o progresso em "checkpoints" (commits).

**1. Crie o diretório do projeto:**

```bash
mkdir elixir_todo_list
cd elixir_todo_list
```

!!! warning
    **O nome da pasta importa!** O Phoenix vai derivar o nome da aplicação (`:elixir_todo_list`) e o prefixo de **todos os módulos** (`ElixirTodoList...`) do nome desta pasta. Se você usar outro nome, terá que adaptar todos os nomes de módulo do tutorial. Recomendamos usar exatamente `elixir_todo_list`.


**2. Inicialize o Git:**

```bash
git init
git branch -m main
```

**3. Crie um `.gitignore` inicial.**

Antes de gerar qualquer código, vamos garantir que o repositório só contenha o que é realmente necessário. O `.gitignore` diz ao Git o que **não** versionar — dependências, arquivos compilados, configurações locais e **o arquivo do banco de dados** (que criaremos na Fase 3: ele é _resultado_ da aplicação, não código-fonte).

Crie o arquivo `.gitignore` (atenção à grafia: **ponto + gitignore**, sem letras faltando!) com o conteúdo:

```gitignore
/_build
/deps
/priv/static/assets
/node_modules
/assets/node_modules
*.log
/config/dev.secret.exs
.DS_Store
.vscode/

# --- Banco de Dados (SQLite, usado a partir da Fase 3) ---
*.db
*.db-shm
*.db-wal
```

### 💾 Passo 0.4: O Commit Inicial

```bash
git add .
git commit -m "Fase 0: Inicializa o repositório e .gitignore"
```

!!! tip
    **Se o Git reclamar de identidade** (`Please tell me who you are`), configure uma vez:

    ```bash
    git config --global user.name "Seu Nome"
    git config --global user.email "seu@email.com"
    ```

    e repita o commit.


### 🧩 Passo 0.5: Gerar o Esqueleto do Projeto

Com o Mix e o gerador Phoenix instalados, vamos criar a estrutura da aplicação **dentro do diretório atual**:

```bash
mix phx.new . --no-ecto
```

**🔍 Analisando o comando:**

- `.` → o projeto será criado **no diretório atual**. (Se quiséssemos uma pasta nova, seria `mix phx.new minha_app`.)
- `--no-ecto` → evita instalar o **Ecto** (a camada de banco de dados) por enquanto. Vamos **adiar essa parte** para a Fase 3, pois queremos primeiro entender o funcionamento **"em memória"** do LiveView.

!!! tip
    E o LiveView? Desde o Phoenix 1.7, o **LiveView já vem incluído por padrão** — não é preciso nenhuma flag para ativá-lo (existe apenas `--no-live` para quem quiser removê-lo, o que não é o nosso caso).


**💡 Comparando com outras stacks:**

- No **Django**: `django-admin startproject nome_do_projeto`.
- No **Rails**: `rails new nome_do_projeto`.
- No **Node.js**: `npx create-next-app` ou `express-generator`.

Assim como nesses casos, o Phoenix gera **um esqueleto completo** de aplicação web, com pastas organizadas para templates, rotas, assets e (opcionalmente) banco de dados.

**⚠️ Atenção às perguntas do gerador!** Como a pasta não está vazia, o Phoenix fará algumas perguntas:

1. `The directory ... already exists. Are you sure you want to continue?` → responda **Y**.
2. `.gitignore already exists, overwrite?` → responda **Y** (sim!). O `.gitignore` do Phoenix é mais completo que o nosso — vamos aceitá-lo e **recolocar nossas regras do banco em seguida**.
3. `Fetch and install dependencies? [Yn]` → responda **Y**.

O Phoenix irá baixar todas as **dependências Elixir** (pelo Hex) e configurar os **assets** (Tailwind/esbuild), deixando o projeto pronto para rodar.

### ✂️ Passo 0.6: Reaplicar as Regras do Banco no `.gitignore`

Como aceitamos o `.gitignore` do Phoenix (que não conhece nosso futuro banco SQLite), abra o `.gitignore` e **acrescente ao final**:

```gitignore
# --- Banco de Dados (SQLite, usado a partir da Fase 3) ---
*.db
*.db-shm
*.db-wal
```

!!! warning
    **Por que os três padrões?** O SQLite, no modo padrão do Ecto, cria três arquivos: o banco em si (`.db`) e dois auxiliares de escrita (`.db-shm` e `.db-wal` — este último pode crescer para centenas de KB!). Nenhum deles deve ir para o GitHub. Esquecer isso é um dos erros mais comuns — e mais feios — em repositórios de alunos.


### 💾 Passo 0.7: O Commit do Esqueleto

Agora que temos a estrutura gerada (e o `.gitignore` completo), vamos versionar o ponto de partida:

```bash
git add .
git commit -m "Fase 0: Gera o esqueleto do Phoenix com LiveView (sem Ecto)"
```

Esse commit marca o início oficial do projeto: um esqueleto Phoenix totalmente funcional, com LiveView configurado, mas sem banco de dados — perfeito para explorar a lógica do LiveView em tempo real.

---

**Fim da Fase 0!** 🏁

---

