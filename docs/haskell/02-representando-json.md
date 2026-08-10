# Representando JSON em Haskell: Módulos e Compilação

## Representando dados JSON em Haskell

Primeiro, crie um novo arquivo `SimpleJSON.hs` em `src/`:

```haskell
-- src/SimpleJSON.hs
module SimpleJSON where
```

Para trabalhar com dados JSON no Haskell, usamos um tipo de dados algébrico para representar os valores possíveis nesse formato:

```haskell
-- src/SimpleJSON.hs
data JValue = JString String
            | JNumber Double
            | JBool Bool
            | JNull
            | JObject [(String, JValue)]
            | JArray [JValue]
              deriving (Eq, Ord, Show)
```

Para cada tipo de JSON, fornecemos um construtor de valor distinto. Alguns desses construtores possuem parâmetros: se quisermos construir uma string JSON, devemos fornecer um valor `String` como argumento para o construtor `JString`.

Para começar a experimentar esse código, salve o arquivo `SimpleJSON.hs` no seu editor, alterne para uma janela de terminal e carregue o projeto no REPL executando o seguinte comando na raiz do projeto:

```
$ stack ghci
Using main module: 1. Package `hs2json' component hs2json:exe:hs2json-exe with main-is file: .../app/Main.hs
Building all executables for `hs2json' once. ...
Configuring GHCi with the following packages: hs2json
GHCi, version 9.4.7: https://www.haskell.org/ghc/  :? for help
[1 of 3] Compiling Lib              ( src/Lib.hs, interpreted )
[2 of 3] Compiling SimpleJSON       ( src/SimpleJSON.hs, interpreted )
[3 of 3] Compiling Main             ( app/Main.hs, interpreted )
Ok, three modules loaded.
ghci> JString "foo"
JString "foo"
ghci> JNumber 2.7
JNumber 2.7
ghci> :type JBool True
JBool True :: JValue
```

_(A saída exata do cabeçalho varia com as versões, mas o prompt e o comportamento são estes.)_

Podemos ver como usar um construtor para pegar um valor Haskell normal e transformá-lo em um `JValue`. Para fazer o inverso, usamos casamento de padrões. Aqui está uma função que podemos adicionar ao `SimpleJSON.hs`, que irá extrair uma string de um valor JSON para nós. Se o valor JSON realmente contiver uma string, nossa função envolverá a string com o construtor `Just`. Caso contrário, retornará `Nothing`.

```haskell
-- src/SimpleJSON.hs
getString :: JValue -> Maybe String
getString (JString s) = Just s
getString _           = Nothing
```

Quando salvamos o arquivo de código-fonte modificado, podemos recarregá-lo no `stack ghci` com o comando `:r` e testar a nova definição:

```
ghci> :r
[2 of 3] Compiling SimpleJSON       ( src/SimpleJSON.hs, interpreted )
Ok, three modules loaded.
ghci> getString (JString "hello")
Just "hello"
ghci> getString (JNumber 3)
Nothing
```

A seguir, mais algumas funções acessoras. Desta vez incluímos as assinaturas de tipo — o GHC as infere sozinho, mas escrevê-las é uma boa prática (e o template do Stack ativa avisos que nos lembram disso):

```haskell
-- src/SimpleJSON.hs
getInt :: JValue -> Maybe Int
getInt (JNumber n) = Just (truncate n)
getInt _           = Nothing

getDouble :: JValue -> Maybe Double
getDouble (JNumber n) = Just n
getDouble _           = Nothing

getBool :: JValue -> Maybe Bool
getBool (JBool b) = Just b
getBool _         = Nothing

getObject :: JValue -> Maybe [(String, JValue)]
getObject (JObject o) = Just o
getObject _           = Nothing

getArray :: JValue -> Maybe [JValue]
getArray (JArray a) = Just a
getArray _          = Nothing

isNull :: JValue -> Bool
isNull v = v == JNull
```

A função `truncate` transforma um número de ponto flutuante ou racional em um inteiro, descartando os dígitos após o ponto decimal:

```
ghci> truncate 5.8
5
ghci> :module +Data.Ratio
ghci> truncate (22 % 7)
3
```

## A anatomia de um módulo Haskell

Um arquivo fonte do Haskell contém a definição de um único _module_. Um módulo nos permite determinar quais nomes dentro dele são acessíveis a partir de outros módulos.

Um arquivo fonte começa com uma declaração `module`. Ela deve preceder todas as outras definições no arquivo:

```haskell
-- src/SimpleJSON.hs
module SimpleJSON
    (
      JValue(..)
    , getString
    , getInt
    , getDouble
    , getBool
    , getObject
    , getArray
    , isNull
    ) where
```

A palavra `module` é reservada. Ela é seguida pelo nome do módulo, que deve começar com uma letra maiúscula. Um arquivo fonte deve ter o mesmo _base name_ (o componente antes do sufixo) que o nome do módulo que ele contém. É por isso que nosso arquivo `SimpleJSON.hs` contém um módulo chamado `SimpleJSON`.

Após o nome do módulo há uma lista de _exportações_, entre parênteses. A palavra-chave `where` indica que o corpo do módulo vem a seguir.

A lista de exportações indica quais nomes deste módulo estão visíveis para outros módulos. Isso nos permite manter o código privado escondido do mundo exterior. A notação especial `(..)` que segue o nome `JValue` indica que estamos exportando o tipo **e todos os seus construtores**.

Pode parecer estranho que possamos exportar o nome de um tipo (isto é, seu construtor de tipo) mas não seus construtores de valor. A capacidade de fazer isso é importante: ela nos permite ocultar os detalhes de um tipo dos seus usuários, tornando o tipo **abstrato**. Se não podemos ver os construtores de valor de um tipo, não podemos casar padrões com um valor desse tipo, nem construir um novo valor desse tipo. Mais adiante neste capítulo, veremos uma situação em que **queremos** exatamente isso.

Se omitirmos as exportações (e os parênteses que as envolvem) da declaração do módulo, todos os nomes do módulo serão exportados:

```haskell
module ExportEverything where
```

Para não exportar nenhum nome (o que raramente é útil), escrevemos uma lista de exportação vazia, usando um par de parênteses:

```haskell
module ExportNothing () where
```

## Compilando um programa Haskell

Para compilar o projeto e executar o binário, na raiz do projeto:

```
$ stack build
$ stack run
someFunc
```

_(O `someFunc` vem do `src/Lib.hs` gerado pelo template — é o "hello world" do esqueleto.)_

Agora que compilamos com sucesso nossa biblioteca mínima, vamos começar a escrever a biblioteca proposta aqui. Antes de seguir, **apague o arquivo `src/Lib.hs`**, já que não o usaremos mais, e então modifique o `app/Main.hs`:

```haskell
-- app/Main.hs
module Main (main) where

import SimpleJSON

main :: IO ()
main = print (JObject [("foo", JNumber 1), ("bar", JBool False)])
```

!!! tip
    Graças ao hpack, apagar `Lib.hs` e criar `SimpleJSON.hs` **não exige editar configuração nenhuma**: no próximo `stack build`, o arquivo `hs2json.cabal` é regenerado refletindo os módulos que existem em `src/`. (No fluxo antigo do livro original, cada módulo novo precisava ser registrado à mão no `.cabal`.)

Observe a diretiva `import` que segue a declaração do módulo. Ela indica que queremos pegar todos os nomes exportados do módulo `SimpleJSON` e disponibilizá-los no nosso módulo. Quaisquer diretivas `import` devem aparecer em grupo, no início do módulo — após a declaração `module`, mas antes de todo o resto do código. Não podemos espalhá-las pelo arquivo.

Os nomes dos arquivos fonte e das funções ficam a cargo do programador. Porém, para criar um executável, o GHC espera um módulo chamado `Main` que contenha uma função chamada `main`. A função `main` é a que será chamada quando executarmos o programa.

```
$ stack build
$ stack run
JObject [("foo",JNumber 1.0),("bar",JBool False)]
```

