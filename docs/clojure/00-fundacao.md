# Fase 0: A Fundação (Ambiente, Setup e Git)

**Objetivo:** Garantir que sua máquina tem todas as ferramentas necessárias, criar a pasta do projeto e iniciar o controle de versão com um `.gitignore` correto.

### Passo 0.1: Verificar o Ambiente (Pré-requisitos)

**Por que fazemos isso?**
Nada é mais frustrante do que descobrir, no meio da Fase 3, que o Node.js não está instalado. Vamos verificar tudo **agora**, em 30 segundos.

**Ação:** Abra seu terminal e execute os comandos abaixo, um por um:

```bash
java -version         # Precisa ser 11 ou superior (17 ou 21 recomendado)
clj --version         # O Clojure CLI (qualquer versão 1.11+).
                      # Se falhar reclamando de rlwrap, use: clojure --version
node -v               # Precisa ser 18 ou superior
git --version         # Qualquer versão recente
```

> [!NOTE]
> **Sobre o clj vs. clojure:**
> O utilitário `clj` é apenas um script wrapper que tenta executar o Clojure em conjunto com a ferramenta de edição de linha de comando `rlwrap`. Em alguns sistemas, se o `rlwrap` não estiver instalado, rodar `clj` resultará em um erro como:
> `Please install rlwrap for command editing or use "clojure" instead.`
> Se for o seu caso, não se preocupe! Você não precisa instalar nada a mais. Basta usar o comando `clojure` no lugar de `clj` em todo este tutorial (ex: `clojure --version` para testar a versão, `clojure -M:run` para rodar o backend e `clojure` para REPL). Ambos executam exatamente o mesmo compilador.

**Resultado Esperado:** Cada comando deve imprimir uma versão. Se algum deles der `command not found`:

- **Java:** instale um JDK (ex: [Adoptium/Temurin](https://adoptium.net/)).
- **Clojure CLI:** siga o guia oficial em [clojure.org/guides/install_clojure](https://clojure.org/guides/install_clojure).
- **Node.js:** instale a versão LTS em [nodejs.org](https://nodejs.org/).
- **Git:** [git-scm.com](https://git-scm.com/).

!!! tip
    O `npm` (gerenciador de pacotes do Node) vem junto com o Node.js. Você pode confirmar com `npm -v`.


### Passo 0.2: Criar a Estrutura de Pastas

Primeiro, vamos criar um diretório principal para o projeto e navegar para dentro dele.

**Ação:** No seu terminal, execute:

```bash
mkdir todo-app
cd todo-app
```

Agora, você deve estar dentro da pasta `todo-app/`. Esta será a "raiz" (root) de todo o nosso projeto, onde colocaremos o `deps.edn`, o `.gitignore` e tudo mais.

!!! warning
    **Todos os comandos deste tutorial são executados a partir desta pasta raiz** (`todo-app/`), a menos que digamos o contrário. Se um comando falhar com "arquivo não encontrado", a primeira coisa a verificar é: *estou na pasta certa?* (use `pwd` para conferir).


### Passo 0.3: Iniciar o Git

**Por que fazemos isso?**
O Git é nosso sistema de controle de versão. Pense nele como uma "máquina do tempo" para o nosso código. Ele nos permite salvar "fotos" (chamadas _commits_) do nosso projeto à medida que avançamos. Se algo quebrar, podemos facilmente voltar para uma versão que funcionava.

**Ação:** Dentro da pasta `todo-app/`, execute:

```bash
git init
git branch -m main
```

**O que vai acontecer?** Você verá uma resposta parecida com:
`Initialized empty Git repository in /path/to/your/todo-app/.git/`

O Git criou um subdiretório oculto chamado `.git`. É ali que ele armazena todo o histórico. Você não precisa (e geralmente não deve) mexer nesse diretório diretamente. O segundo comando apenas renomeia a branch principal para `main` (o padrão moderno).

### Passo 0.4: Criar o `.gitignore`

**Por que fazemos isso?**
O Git agora está observando _tudo_ em sua pasta. Mas não queremos salvar tudo. Existem arquivos que **não** devem ir para o controle de versão:

1. **Dependências:** pastas como `node_modules/` podem ter milhares de arquivos. Elas podem ser reinstaladas a qualquer momento.
2. **Arquivos compilados:** o Clojure e o shadow-cljs criam pastas de "saída" (como `target/` ou `resources/public/js/`). Nosso código-fonte é o que importa; o código compilado é apenas um resultado.
3. **Arquivos de sistema/IDE:** seu sistema operacional ou sua IDE podem criar "lixo" (como `.DS_Store` ou `.calva/`).
4. **Dados gerados pela aplicação:** na Fase 5, nossa aplicação criará um arquivo de banco de dados (`prod.db`). Ele é _resultado_ da aplicação rodando, não código-fonte — já vamos deixá-lo ignorado desde agora.
5. **Segredos:** se um dia você tiver uma chave de API ou senha, ela também iria para o `.gitignore` para _nunca_ ser enviada ao GitHub.

**Ação:** Crie um novo arquivo na raiz do projeto chamado `.gitignore` (começando com um ponto) e cole o seguinte conteúdo:

```gitignore
# --- Geral ---
# Arquivos de sistema operacional
.DS_Store
Thumbs.db
*.log

# --- Dependências ---
# Dependências do Node.js (para shadow-cljs)
/node_modules/

# --- Clojure & Java ---
# Pasta de build padrão
/target/

# Cache de dependências do Clojure CLI
.cpcache/
.clj-kondo/
.cider-repl-history

# --- shadow-cljs (Frontend) ---
# Saída do build do frontend
/resources/public/js/

# Cache do shadow-cljs
.shadow-cljs/

# --- IDEs ---
# VS Code
.vscode/

# Emacs
*~
\#*\#

# Calva (VS Code Clojure)
.calva/

# --- Banco de Dados (usado a partir da Fase 5) ---
*.db
*.sqlite
*.sqlite3
```

### O que fizemos?

Instruímos o Git a ignorar as pastas e arquivos mais comuns de um projeto Clojure/ClojureScript — **incluindo, desde já, o arquivo do banco de dados** que só aparecerá na Fase 5. Assim ninguém commita o `prod.db` por acidente.

Agora, quando você executar `git status`, verá seu novo arquivo `.gitignore`, mas não verá nenhuma das pastas listadas (mesmo que elas existam).

### Passo 0.5: O Primeiro Commit

**Por que fazemos isso?**
Até agora, criamos um arquivo (`.gitignore`) e o Git sabe que ele existe, mas ele não foi "salvo" na nossa linha do tempo.

O processo no Git é sempre em duas etapas:

1. **Stage (Preparar):** você diz ao Git quais arquivos quer incluir na próxima "foto". O comando é `git add`.
2. **Commit (Salvar):** você tira a "foto" de todos os arquivos preparados e anexa uma mensagem. O comando é `git commit`.

**Ação:** Execute estes dois comandos, um após o outro:

```bash
# 1. Adiciona TODOS os arquivos novos ou modificados na área de "Stage"
#    (neste caso, apenas o .gitignore)
git add .

# 2. Salva (faz o commit) os arquivos que estão em "Stage"
#    -m "..." é a mensagem que descreve o que fizemos
git commit -m "feat: setup inicial do projeto com .gitignore"
```

**Resultado Esperado:**

```
[main (root-commit) a1b2c3d] feat: setup inicial do projeto com .gitignore
 1 file changed, 40 insertions(+)
 create mode 100644 .gitignore
```

!!! tip
    **Se o Git reclamar de identidade** (`Please tell me who you are`), configure uma vez:

    ```bash
    git config --global user.name "Seu Nome"
    git config --global user.email "seu@email.com"
    ```

    e repita o `git commit`.


### O que fizemos?

Salvamos a "Versão Zero" do nosso projeto. Você pode executar `git log` a qualquer momento para ver o histórico.

**Sobre a Mensagem de Commit (`feat: ...`):**
A mensagem `feat: setup inicial...` segue uma convenção chamada **Conventional Commits**:

- `feat:` significa _feature_ (uma nova funcionalidade — neste caso, o próprio setup).
- Outros prefixos comuns: `fix:` (corrige um bug), `docs:` (documentação), `refactor:` (muda o código sem mudar o comportamento) e `style:` (formatação/visual).
- Usar isso torna seu histórico Git muito fácil de ler — e é **exatamente o que será avaliado** no seu repositório.

---

**Fim da Fase 0!** 🏁

Temos uma fundação sólida: ambiente verificado, uma pasta de projeto limpa, um repositório Git rastreando nossas mudanças e um `.gitignore` para manter o "lixo" do lado de fora.

---

