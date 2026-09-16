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

A comunidade Haskell padronizou a descrição de software no formato **Cabal**: cada _pacote_ contém uma biblioteca e, possivelmente, executáveis, descritos em um arquivo `.cabal`. É esse o formato que o Hackage (o repositório central de pacotes) e todas as ferramentas entendem — e, como vimos no Capítulo 1, é também o **único** arquivo de configuração do nosso projeto: sem a camada `package.yaml`/hpack de outras ferramentas. Vamos completar o `hs2json.cabal` que o `cabal init` gerou, entendendo cada seção.

### A descrição do pacote

Abra o `hs2json.cabal`. A primeira parte são as propriedades globais do pacote:

```cabal
cabal-version:   3.0
name:            hs2json
version:         0.1.0.0
license:         BSD-3-Clause
author:          Seu Nome
maintainer:      seu@email.org
```

Nomes de pacotes devem ser **únicos** dentro do seu conjunto de dependências (e globalmente, se um dia você publicar no Hackage). A versão segue a PVP (_Package Versioning Policy_), a política de versionamento do ecossistema.

Boa parte das propriedades destina-se a leitores humanos, não às ferramentas:

```cabal
synopsis:        Minha biblioteca de impressão agradável, com suporte a JSON
description:     Uma pequena biblioteca de pretty printing que ilustra
                 como desenvolver uma biblioteca Haskell.
category:        Text
```

A maioria dos pacotes Haskell usa a licença BSD de 3 cláusulas, que o Cabal chama de `BSD-3-Clause` (você é livre para escolher a que achar apropriada; o campo `license-file` aponta para o arquivo com o texto exato).

Em seguida vêm as seções `library` e `executable`, cada uma com seus próprios `build-depends` e `exposed-modules`:

```cabal
library
    exposed-modules: SimpleJSON, PutJSON, Prettify, PrettyJSON, QuickTestes
    hs-source-dirs:  src
    build-depends:   base >= 4.7 && < 5
    default-language: Haskell2010

executable hs2json-exe
    main-is:         Main.hs
    hs-source-dirs:  app
    build-depends:   base >= 4.7 && < 5, hs2json
    default-language: Haskell2010
```

Traduzindo:

- **`build-depends`** lista os pacotes de que precisamos, com faixas de versão. Nossa biblioteca só usa o `base` (que traz o Prelude, `Data.Bits`, `Numeric` etc.).
- **`exposed-modules`** lista, um a um, os módulos que compõem a biblioteca. Diferente de ferramentas com detecção automática, o Cabal **não varre** o diretório `src/` sozinho: cada módulo novo — `Prettify`, `PrettyJSON`, `PutJSON`, `SimpleJSON` — precisa ser acrescentado à mão a essa lista, ou o `cabal build` não vai enxergá-lo. (Se um dia você quiser módulos **internos**, invisíveis aos usuários do pacote, declare-os em `other-modules:` em vez de `exposed-modules:`.)
- **`executable`** descreve o binário. Note que ele **depende da própria biblioteca** (`hs2json`) — é assim que o `Main.hs` enxerga o `SimpleJSON`.

!!! note
    **Entendendo as dependências:** não precisamos adivinhar quais pacotes declarar. Experimente remover a linha `base >= 4.7 && < 5` e rodar `cabal build`: a compilação falha imediatamente, com o GHC dizendo que não encontra nem o Prelude. A mensagem de erro nos diz o que falta — recoloque a linha e tudo volta. Explicitar as dependências tem um benefício prático enorme: é o que permite ao cabal-install baixar, compilar e instalar automaticamente **tudo** de que um pacote precisa, recursivamente.

### Como o Cabal resolve versões

Diferente de ferramentas baseadas em _snapshots_ (um conjunto fixo de versões de pacotes testadas juntas), o cabal-install resolve as versões das dependências contra o **índice completo do Hackage**, respeitando as faixas de versão (`>= 4.7 && < 5`) que você declarou em cada `build-depends`. Isso te dá acesso imediato a qualquer versão publicada de qualquer pacote, ao custo de builds um pouco menos reprodutíveis entre máquinas diferentes por padrão — se isso for uma preocupação (por exemplo, numa disciplina, para garantir que o projeto de todo mundo compile igual), o comando `cabal freeze` grava um arquivo `cabal.project.freeze` fixando a versão exata resolvida de cada dependência, para todo mundo usar a mesma.

### Compilando, testando e instalando

Com a descrição pronta, o ciclo completo é:

```
$ cabal build            # compila biblioteca e executáveis
$ cabal run hs2json-exe  # executa o executável
$ cabal test             # roda a suíte de testes (test/Spec.hs)
$ cabal install           # copia o executável para ~/.local/bin (ou ~/.cabal/bin)
```

O `cabal install` deixa o binário disponível no seu `PATH` (se o diretório de instalação estiver nele) — é o equivalente moderno do antigo `runghc Setup install`, sem nenhuma configuração prévia.

### E o Stack?

Tudo que fizemos tem equivalente direto na outra ferramenta popular do ecossistema, o **Stack**: `stack new` cria o projeto (gerando um `package.yaml`, que uma ferramenta embutida chamada hpack converte em `.cabal` a cada build — com a vantagem de detectar módulos novos em `src/` sozinha, sem precisar listá-los à mão), e `stack build` / `stack run` / `stack test` / `stack install` espelham os comandos do Cabal que já vimos. A diferença prática mais relevante: o Stack resolve dependências contra um _snapshot_ do Hackage (um `resolver`, declarado no `stack.yaml`, testado como um conjunto coeso) em vez do índice completo — o que tende a dar builds mais reprodutíveis entre máquinas diferentes sem precisar de um passo extra como o `cabal freeze`. Saber que as duas ferramentas falam o mesmo formato `.cabal` por baixo é o que importa: o conhecimento deste capítulo vale para as duas.

## Dicas práticas e leitura adicional

O ecossistema tem bibliotecas de impressão agradável prontas e maduras — recomendamos usá-las em código real, em vez de escrever a sua:

- **[prettyprinter](https://hackage.haskell.org/package/prettyprinter)** é a biblioteca moderna de referência, com anotações (por exemplo, para saída colorida) e uma API muito próxima da que construímos: você reconhecerá `<>`, `group`, `nest`, `softline` na hora.
- **`Text.PrettyPrint.HughesPJ`** (pacote `pretty`, distribuído com o GHC) é a biblioteca clássica citada no livro original, ainda amplamente usada.

O design dessas bibliotecas tem história: a HughesPJ foi introduzida por John Hughes em _The Design of a Pretty-Printing Library_ (1995) e melhorada por Simon Peyton Jones — daí o nome. A nossa, como a do livro, é baseada no sistema mais simples descrito por Philip Wadler em _A Prettier Printer_ (1998), estendido por Daan Leijen na antiga `wl-pprint` — da qual a `prettyprinter` moderna é a sucessora direta. O artigo do Hughes é longo, mas vale a leitura pela discussão de como **projetar** uma biblioteca em Haskell — que foi, afinal, o verdadeiro assunto deste capítulo.

---

