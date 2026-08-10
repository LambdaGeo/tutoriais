# Imprimindo e Renderizando Dados JSON

## Imprimindo dados JSON

Agora que temos uma representação em Haskell para os tipos JSON, gostaríamos de ser capazes de pegar valores Haskell e produzi-los como dados JSON.

Há algumas maneiras de fazer isso. Talvez a mais direta seja escrever uma função que imprima os valores no formato JSON. Quando terminarmos, exploraremos abordagens mais interessantes.

```haskell
-- src/PutJSON.hs
module PutJSON where

import Data.List (intercalate)
import SimpleJSON

renderJValue :: JValue -> String
renderJValue (JString s)   = show s
renderJValue (JNumber n)   = show n
renderJValue (JBool True)  = "true"
renderJValue (JBool False) = "false"
renderJValue JNull         = "null"
renderJValue (JObject o) = "{" ++ pairs o ++ "}"
  where pairs [] = ""
        pairs ps = intercalate ", " (map renderPair ps)
        renderPair (k,v) = show k ++ ": " ++ renderJValue v
renderJValue (JArray a) = "[" ++ values a ++ "]"
  where values [] = ""
        values vs = intercalate ", " (map renderJValue vs)
```

Uma boa prática em Haskell é separar o código puro do código que produz efeitos de entrada e saída (`IO ()`). Nossa função `renderJValue` não interage com o mundo exterior, mas ainda precisamos ser capazes de **imprimir** um `JValue`:

```haskell
-- src/PutJSON.hs
putJValue :: JValue -> IO ()
putJValue v = putStrLn (renderJValue v)
```

Imprimir um valor JSON agora é fácil.

Por que separar o código de renderização do código que realmente imprime? Isso nos dá flexibilidade. Por exemplo: se quiséssemos compactar os dados antes de imprimi-los, e o código de renderização estivesse misturado com o de impressão, adaptar o código seria muito mais difícil.

Essa ideia de separar código puro de código impuro é poderosa e onipresente em Haskell. Várias bibliotecas de compressão existem, e todas têm uma interface simples: uma função que aceita uma string descompactada e retorna uma string compactada. Podemos usar composição de funções para converter dados JSON em string e compactá-los em outra string, adiando qualquer decisão sobre como, efetivamente, mostrar ou transmitir os dados.

Experimentando:

```
$ stack ghci
ghci> import PutJSON
ghci> putJValue (JObject [("nome", JString "Sergio"), ("idade", JNumber 38)])
{"nome": "Sergio", "idade": 38.0}
```

## Uma visão mais geral de renderização

Nosso código de renderização JSON está adaptado às necessidades dos nossos tipos de dados e às convenções de formatação do JSON. A saída que ele produz pode não ser amigável aos olhos humanos. Agora olharemos a renderização como uma tarefa mais genérica: como construir uma biblioteca útil para renderizar dados em uma variedade de situações?

Gostaríamos de produzir saídas adequadas tanto para consumo humano (para depurar, por exemplo) quanto para processamento por máquinas. Bibliotecas que fazem essa tarefa são chamadas de _pretty printers_ — "impressoras agradáveis". Há várias bibliotecas Haskell prontas desse tipo. Não estamos criando a nossa para substituí-las, mas pelos vários aprendizados que ganharemos em design de bibliotecas e técnicas de programação funcional.

Chamaremos nosso módulo genérico de pretty printing de `Prettify`; o código ficará no arquivo `src/Prettify.hs`.

!!! note
    **Nomeando:** no módulo `Prettify`, basearemos nossos nomes naqueles usados por várias bibliotecas bem estabelecidas desse tipo. Isso nos dá um grau de compatibilidade com as bibliotecas maduras.

Para termos certeza de que `Prettify` atende a necessidades práticas, escreveremos um novo renderizador de JSON que usa a API do `Prettify`. Depois que estiver pronto, voltaremos e preencheremos os detalhes do módulo `Prettify`.

Em vez de renderizar direto para string, nosso `Prettify` usará um tipo **abstrato**, que chamaremos de `Doc`. Baseando nossa biblioteca em um tipo abstrato, podemos trocar a implementação por uma mais flexível ou mais eficiente sem que os usuários da biblioteca percebam.

Chamaremos nosso novo renderizador JSON de `PrettyJSON.hs`, mantendo o nome `renderJValue` para a função de renderização. Renderizar um dos valores básicos do JSON é simples:

```haskell
-- src/PrettyJSON.hs
module PrettyJSON where

import Prelude hiding ((<>))

import SimpleJSON
import Prettify

renderJValue :: JValue -> Doc
renderJValue (JBool True)  = text "true"
renderJValue (JBool False) = text "false"
renderJValue JNull         = text "null"
renderJValue (JNumber num) = double num
renderJValue (JString str) = string str
```

O tipo `Doc` e as funções `text`, `double` e `string` serão fornecidos pelo nosso módulo `Prettify`.

!!! warning
    **A linha `import Prelude hiding ((<>))` é obrigatória — e é a maior mudança desde o livro original.** Quando _Real World Haskell_ foi escrito, o operador `<>` era um nome livre. Desde o **GHC 8.4** (2018), porém, o Prelude exporta `<>` (o operador da classe `Semigroup`). Como nossa biblioteca define o **seu próprio** `<>` para concatenar documentos, precisamos esconder o do Prelude em **todos os módulos que definem ou usam o nosso** — ou seja, tanto em `Prettify.hs` quanto em `PrettyJSON.hs`. Se você esquecer essa linha, o GHC reclamará de _"Ambiguous occurrence '<>'"_. Guarde este erro: ele é um clássico ao seguir material antigo de Haskell.

## Desenvolvendo código Haskell sem quebrar a cabeça

No início, quando estamos nos familiarizando com o desenvolvimento em Haskell, temos tantos conceitos novos para entender de uma vez que escrever código que compile sem erros pode ser um desafio.

Enquanto escrevemos o corpo inicial do código, ajuda muito parar a cada poucos minutos e tentar compilar o que produzimos até o momento. Como Haskell é fortemente tipado, se o código compila, estamos longe de muitas armadilhas da programação.

Uma técnica útil para desenvolver o esqueleto de um programa é escrever versões _de esboço_ (placeholders) dos nossos tipos e funções. Por exemplo: dissemos acima que as funções `string`, `text` e `double` serão fornecidas pelo módulo `Prettify`. Se não fornecermos definições para essas funções nem para o tipo `Doc`, nosso lema "compile cedo, compile frequentemente" falha logo no renderizador, pois o compilador não conhece nada sobre elas. Para evitar o problema, escrevemos esboços que não fazem nada:

```haskell
-- src/Prettify.hs
module Prettify where

import Prelude hiding ((<>))

data Doc = ToBeDefined
         deriving (Show)

string :: String -> Doc
string str = undefined

text :: String -> Doc
text str = undefined

double :: Double -> Doc
double num = undefined
```

O valor especial `undefined` tem o tipo `a`, então ele passa pela verificação de tipos não importa onde o usemos. Se tentarmos **avaliá-lo**, ele causará um erro no programa:

```
ghci> :type undefined
undefined :: HasCallStack => a
ghci> undefined
*** Exception: Prelude.undefined
ghci> :type double
double :: Double -> Doc
ghci> double 3.14
*** Exception: Prelude.undefined
```

_(No GHC moderno, o tipo aparece como `HasCallStack => a` — o `HasCallStack` é só o mecanismo que permite ao erro apontar a linha exata onde o `undefined` explodiu. Para nossos propósitos, leia como o `a` do livro original.)_

Embora não possamos **executar** os esboços, o verificador de tipos garante que o programa está sensatamente tipado.

