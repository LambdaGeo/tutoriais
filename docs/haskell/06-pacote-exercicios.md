# Exercícios, Empacotamento e Leitura Adicional

## Exercícios

Nossa biblioteca de impressão agradável é concisa — de modo a caber nas restrições de espaço de um capítulo —, mas há várias melhorias úteis que podemos fazer.

**1.** Escreva a função `fill`, com a seguinte assinatura de tipos:

```haskell
fill :: Int -> Doc -> Doc
```

Ela deve adicionar espaços a um documento até que ele atinja a largura dada em colunas. Se o documento já é mais largo que isso, ela não adiciona nada.

**2.** Nosso `Prettify` não leva **indentação** em conta. Quando abrimos parênteses, chaves ou colchetes, as linhas seguintes deveriam ser indentadas, alinhadas com o caractere de abertura, até o caractere de fechamento correspondente. Adicione suporte a indentação, com quantidade controlável de espaços:

```haskell
nest :: Int -> Doc -> Doc
```

## Criando um pacote

_(Esta seção foi inteiramente reescrita: o fluxo original — `Setup.hs`, `runghc Setup configure` e `ghc-pkg` — pertence à era pré-2010 do Cabal e não é mais como se trabalha.)_

A comunidade Haskell padronizou a descrição de software no formato **Cabal**: cada _pacote_ contém uma biblioteca e, possivelmente, executáveis, descritos em um arquivo `.cabal`. É esse o formato que o Hackage (o repositório central de pacotes) e todas as ferramentas entendem.

Como vimos no início, nosso projeto tem uma camada de conveniência por cima disso: o **`package.yaml`**, que o hpack converte em `hs2json.cabal` a cada build. Vamos entender o que há nele — os conceitos são os mesmos do `.cabal`, só que em YAML.

### A descrição do pacote

Abra o `package.yaml`. A primeira parte são as propriedades globais do pacote:

```yaml
name: hs2json
version: 0.1.0.0
license: BSD-3-Clause
author: "Seu Nome"
maintainer: "seu@email.org"
```

Nomes de pacotes devem ser **únicos** dentro do seu conjunto de dependências (e globalmente, se um dia você publicar no Hackage). A versão segue a PVP (_Package Versioning Policy_), a política de versionamento do ecossistema.

Boa parte das propriedades destina-se a leitores humanos, não às ferramentas:

```yaml
synopsis: Minha biblioteca de impressão agradável, com suporte a JSON
description: Uma pequena biblioteca de pretty printing que ilustra
  como desenvolver uma biblioteca Haskell.
category: Text
```

A maioria dos pacotes Haskell usa a licença BSD de 3 cláusulas, que o Cabal chama de `BSD-3-Clause` (você é livre para escolher a que achar apropriada; o campo `license-file` aponta para o arquivo com o texto exato).

Em seguida vêm as **dependências** e os componentes. No template, as dependências valem para todos os componentes:

```yaml
dependencies:
  - base >= 4.7 && < 5

library:
  source-dirs: src

executables:
  hs2json-exe:
    main: Main.hs
    source-dirs: app
    dependencies:
      - hs2json
```

Traduzindo:

- **`dependencies`** lista os pacotes de que precisamos, com faixas de versão. Nossa biblioteca só usa o `base` (que traz o Prelude, `Data.Bits`, `Numeric` etc.).
- **`library`** descreve a biblioteca: todo módulo em `src/` faz parte dela. No `.cabal` gerado, isso vira um campo `exposed-modules:` listando `Prettify`, `PrettyJSON`, `PutJSON` e `SimpleJSON` — o hpack preenche a lista sozinho, varrendo o diretório. (Se um dia você quiser módulos **internos**, invisíveis aos usuários do pacote, declare-os em `other-modules:` no `package.yaml`; tudo que não estiver lá continua exposto.)
- **`executables`** descreve os binários. Note que o executável **depende da própria biblioteca** (`hs2json`) — é assim que o `Main.hs` enxerga o `SimpleJSON`.

!!! note
    **Entendendo as dependências:** não precisamos adivinhar quais pacotes declarar. Experimente remover a linha `- base >= 4.7 && < 5` e rodar `stack build`: a compilação falha imediatamente, com o GHC dizendo que não encontra nem o Prelude. A mensagem de erro nos diz o que falta — recoloque a linha e tudo volta. Explicitar as dependências tem um benefício prático enorme: é o que permite ao Stack (e ao cabal-install) baixar, compilar e instalar automaticamente **tudo** de que um pacote precisa, recursivamente.

### O papel do `stack.yaml` (e onde foi parar o `ghc-pkg`)

No fluxo antigo, o GHC mantinha um banco de dados global de pacotes instalados, manipulado com `ghc-pkg` — e instalar duas versões conflitantes era receita para o infame _"Cabal hell"_. O Stack resolveu isso com os **snapshots**: o `stack.yaml` do projeto aponta para um _resolver_ (por exemplo, `lts-23.x`), que é um conjunto congelado de milhares de pacotes do Hackage **testados juntos**, amarrado a uma versão exata do GHC. Dois projetos com resolvers diferentes convivem sem se tocar. Você raramente precisará editar este arquivo; quando precisar de um pacote fora do snapshot, é nele que se declara (campo `extra-deps`).

### Compilando, testando e instalando

Com a descrição pronta, o ciclo completo é:

```
$ stack build            # compila biblioteca e executáveis
$ stack run              # executa o hs2json-exe
$ stack test             # roda a suíte de testes (test/Spec.hs)
$ stack install          # copia o executável para ~/.local/bin
```

O `stack install` deixa o binário disponível no seu `PATH` (se `~/.local/bin` estiver nele) — é o equivalente moderno do antigo `runghc Setup install`, sem nenhuma configuração prévia.

### E o cabal-install?

Tudo que fizemos tem equivalente direto na outra ferramenta oficial, o **cabal-install**: `cabal init` cria o projeto (gerando o `.cabal` diretamente, sem `package.yaml`), e `cabal build` / `cabal run` / `cabal repl` / `cabal install` espelham os comandos do Stack. As diferenças práticas: o cabal-install resolve versões contra o Hackage inteiro (em vez de snapshots) e usa o GHC que estiver no PATH (instalado pelo GHCup). Para uma disciplina, o Stack tende a dar builds mais reprodutíveis entre as máquinas dos alunos; mas saber que os dois falam o mesmo formato `.cabal` é o que importa.

## Dicas práticas e leitura adicional

O ecossistema tem bibliotecas de impressão agradável prontas e maduras — recomendamos usá-las em código real, em vez de escrever a sua:

- **[prettyprinter](https://hackage.haskell.org/package/prettyprinter)** é a biblioteca moderna de referência, com anotações (por exemplo, para saída colorida) e uma API muito próxima da que construímos: você reconhecerá `<>`, `group`, `nest`, `softline` na hora.
- **`Text.PrettyPrint.HughesPJ`** (pacote `pretty`, distribuído com o GHC) é a biblioteca clássica citada no livro original, ainda amplamente usada.

O design dessas bibliotecas tem história: a HughesPJ foi introduzida por John Hughes em _The Design of a Pretty-Printing Library_ (1995) e melhorada por Simon Peyton Jones — daí o nome. A nossa, como a do livro, é baseada no sistema mais simples descrito por Philip Wadler em _A Prettier Printer_ (1998), estendido por Daan Leijen na antiga `wl-pprint` — da qual a `prettyprinter` moderna é a sucessora direta. O artigo do Hughes é longo, mas vale a leitura pela discussão de como **projetar** uma biblioteca em Haskell — que foi, afinal, o verdadeiro assunto deste capítulo.

---

