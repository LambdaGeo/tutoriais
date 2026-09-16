# Tour pelo JSON e Preparação do Ambiente

*Parte 1 — Escrevendo uma biblioteca para dados no formato JSON*

## Um tour rápido pelo JSON

Neste capítulo, vamos desenvolver uma pequena, mas completa, biblioteca Haskell. Nossa biblioteca manipulará e serializará dados em um popular formato conhecido como JSON.

A linguagem JSON (JavaScript Object Notation) é uma representação pequena e simples para armazenar e transmitir dados estruturados, por exemplo, por meio de uma conexão de rede. É mais comumente usada para transferir dados de um serviço da Web para um aplicativo JavaScript baseado em navegador. O formato JSON é descrito em [www.json.org](https://www.json.org), e em maior detalhe pela RFC 8259 (que substituiu a antiga RFC 4627).

O JSON suporta quatro tipos básicos de valor: _strings_, _numbers_, _booleans_ e um valor especial chamado `null`.

```json
"a string"  12345  true  null
```

A linguagem fornece dois tipos compostos: um _array_ é uma sequência ordenada de valores, e um _object_ é uma coleção não ordenada de pares nome/valor. Os nomes em um objeto são sempre strings; os valores em um objeto ou array podem ser de qualquer tipo.

```json
[-3.14, true, null, "a string"]
{"numbers": [1,2,3,4,5], "useful": false}
```

## Preparando o ambiente: GHCup e Cabal

_(Esta seção substitui o antigo "tour rápido pelo Stack", refletindo o fluxo de instalação atual.)_

A forma recomendada de instalar Haskell hoje é o **[GHCup](https://www.haskell.org/ghcup/)**, o instalador oficial da plataforma. Ele instala e gerencia as versões de todas as ferramentas que precisamos:

| Ferramenta        | O que é                                                                                             |
| ----------------- | --------------------------------------------------------------------------------------------------- |
| **GHC**           | O compilador de Haskell (Glasgow Haskell Compiler).                                                 |
| **cabal-install** | A ferramenta de build e projetos, distribuída junto com o GHC — a que usaremos neste capítulo.      |
| **Stack**         | Outra ferramenta de build e projetos, com foco em builds reprodutíveis via snapshots; alternativa ao Cabal (não usada aqui). |
| **HLS**           | O Haskell Language Server, que dá autocompletar e erros em tempo real no VS Code e outros editores. |

**Instalação (Linux/macOS/WSL):**

```bash
curl --proto '=https' --tlsv1.2 -sSf https://get-ghcup.haskell.org | sh
```

O instalador é interativo — aceite as opções padrão. Quando perguntado sobre o Stack, pode responder **não**: não vamos usá-lo neste guia. Confirme a instalação do **HLS** quando perguntado. No **Windows**, siga as instruções da página do GHCup (há um comando PowerShell equivalente).

**Verificação:** feche e reabra o terminal, e confirme:

```bash
ghc --version      # The Glorious Glasgow Haskell Compilation System, version 9.x
cabal --version    # cabal-install version 3.x ou superior
```

!!! tip
    **Sobre versões:** este capítulo foi validado com GHC 9.4 e GHC 9.10, e o código funciona em qualquer GHC da série 9.x.

!!! warning "No Linux: uma biblioteca do sistema"
    O Cabal compila dependências que precisam de aritmética de precisão arbitrária (GMP) para linkar. Se a compilação falhar com um erro do tipo `cannot find -lgmp`, falta o pacote de desenvolvimento do GMP no seu sistema — no Debian/Ubuntu:
    ```bash
    sudo apt install libgmp-dev
    ```
    O runtime do GHC já depende do GMP, então normalmente já está instalado; falta especificamente o pacote `-dev` com os símbolos de link.

!!! tip "Se o `cabal build`/`cabal update` falhar com erro de assinatura"
    Versões de `cabal-install` muito antigas (por exemplo, as empacotadas pelo `apt` de distribuições Linux mais velhas) às vezes não conseguem validar o índice atual do Hackage, e falham com uma mensagem parecida com `<repo>/root.json does not have enough signatures signed with the appropriate keys`. Isso acontece quando o Hackage rotaciona as chaves de assinatura do índice e o `cabal-install` instalado é velho demais para reconhecer as novas. A correção é instalar um `cabal-install` atual via GHCup (como fizemos acima) em vez de depender do pacote do sistema operacional.

### Criando o projeto

Vamos criar o esqueleto do projeto deste capítulo com o assistente interativo do Cabal:

```bash
mkdir hs2json && cd hs2json
cabal init --interactive
```

Ele faz uma série de perguntas — nome do pacote (`hs2json`), versão, se você quer uma biblioteca, um executável e uma suíte de testes (responda **sim** para os três), licença, linguagem. Ao final, a estrutura gerada é:

```
hs2json/
├── hs2json.cabal     👈 a descrição do pacote (nome, versão, dependências, módulos)
├── src/
│   └── MyLib.hs       👈 a biblioteca (código reutilizável)
├── app/
│   └── Main.hs         👈 o executável (o programa em si)
└── test/
    └── Main.hs           👈 testes (não usaremos neste capítulo)
```

!!! tip
    **Sem `package.yaml`:** o `.cabal` é o único arquivo de configuração — sem a camada `package.yaml`/hpack de outras ferramentas. Isso tem um efeito prático importante: **o Cabal não detecta módulos novos sozinho**. Quando criarmos `SimpleJSON.hs`, `Prettify.hs` etc. dentro de `src/`, precisaremos adicionar cada um à lista `exposed-modules:` do `hs2json.cabal` manualmente — veremos isso já no próximo capítulo.

Os três comandos que usaremos o tempo todo:

```bash
cabal build          # compila o projeto
cabal run             # compila (se preciso) e executa o executável
cabal repl            # abre o REPL com os módulos do projeto carregados
```

Na **primeira** execução de `cabal build`, o Cabal busca o índice de pacotes do Hackage (se ainda não tiver feito isso — `cabal update`) e baixa as dependências — pode demorar um pouco.

