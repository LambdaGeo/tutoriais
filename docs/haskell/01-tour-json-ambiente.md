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

## Preparando o ambiente: GHCup e Stack

_(Esta seção substitui o antigo "tour rápido pelo Stack", refletindo o fluxo de instalação atual.)_

A forma recomendada de instalar Haskell hoje é o **[GHCup](https://www.haskell.org/ghcup/)**, o instalador oficial da plataforma. Ele instala e gerencia as versões de todas as ferramentas que precisamos:

| Ferramenta        | O que é                                                                                             |
| ----------------- | --------------------------------------------------------------------------------------------------- |
| **GHC**           | O compilador de Haskell (Glasgow Haskell Compiler).                                                 |
| **Stack**         | Ferramenta de build e projetos, com versões reprodutíveis (usaremos neste capítulo).                |
| **cabal-install** | A ferramenta de build "clássica"; alternativa ao Stack (falaremos dela ao final).                   |
| **HLS**           | O Haskell Language Server, que dá autocompletar e erros em tempo real no VS Code e outros editores. |

**Instalação (Linux/macOS/WSL):**

```bash
curl --proto '=https' --tlsv1.2 -sSf https://get-ghcup.haskell.org | sh
```

O instalador é interativo — aceite as opções padrão e confirme a instalação do Stack e do HLS quando perguntado. No **Windows**, siga as instruções da página do GHCup (há um comando PowerShell equivalente).

**Verificação:** feche e reabra o terminal, e confirme:

```bash
ghc --version      # The Glorious Glasgow Haskell Compilation System, version 9.x
stack --version    # Version 2.x ou superior
```

!!! tip
    **Sobre versões:** este capítulo foi validado com GHC 9.4, e o código funciona em qualquer GHC da série 9.x. O Stack cuida de fixar uma versão exata de GHC por projeto (via _resolver_), então diferentes projetos podem usar diferentes GHCs sem conflito.

### Criando o projeto

Vamos criar o esqueleto do projeto deste capítulo:

```bash
stack new hs2json
cd hs2json
```

O `stack new` gera uma estrutura de projeto completa. As partes que nos interessam:

```
hs2json/
├── package.yaml      👈 a descrição do pacote (nome, versão, dependências)
├── stack.yaml        👈 configuração do Stack (qual snapshot/GHC usar)
├── src/
│   └── Lib.hs        👈 a biblioteca (código reutilizável)
├── app/
│   └── Main.hs       👈 o executável (o programa em si)
└── test/
    └── Spec.hs       👈 testes (não usaremos neste capítulo)
```

!!! tip
    **`package.yaml` vs `.cabal`:** o formato "oficial" de descrição de pacotes Haskell é o arquivo `.cabal`. O template do Stack usa uma camada mais amigável por cima dele: o `package.yaml`, processado por uma ferramenta chamada **hpack** (embutida no Stack). A cada `stack build`, o hpack gera o arquivo `hs2json.cabal` automaticamente a partir do `package.yaml`. A grande vantagem para nós: o hpack **detecta sozinho os módulos** dentro de `src/` — quando criarmos `SimpleJSON.hs`, `Prettify.hs` etc., não precisaremos registrá-los manualmente em lugar nenhum. (O `.cabal` gerado não deve ser editado à mão; falaremos mais sobre ele na seção de empacotamento, ao final.)

Os três comandos que usaremos o tempo todo:

```bash
stack build          # compila o projeto
stack run            # compila (se preciso) e executa o executável
stack ghci           # abre o REPL com os módulos do projeto carregados
```

Na **primeira** execução de `stack build`, o Stack pode baixar a versão de GHC definida no `stack.yaml` — é demorado, mas acontece uma vez só.

