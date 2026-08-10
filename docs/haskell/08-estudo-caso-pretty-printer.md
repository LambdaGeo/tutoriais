# Estudo de Caso: Especificando um Pretty Printer

## Estudo de caso: especificando um pretty printer

Testar propriedades naturais de funções individuais é uma das abordagens mais básicas que guiam o desenvolvimento de grandes sistemas em Haskell. Veremos agora um cenário mais complicado: construir uma suíte de testes para a biblioteca de _pretty printing_\* da Parte 1.

\*_N. do T.: pretty printing é o nome que se dá à apresentação de um conteúdo de maneira que a estrutura da apresentação reforce o sentido do próprio conteúdo._

### Gerando dados de teste

Lembre-se de que o pretty printer é construído em torno do `Doc`, um tipo de dados algébrico que representa documentos bem estruturados:

```haskell
-- src/Prettify.hs
data Doc = Empty
         | Char Char
         | Text String
         | Line
         | Concat Doc Doc
         | Union Doc Doc
           deriving (Show, Eq)
```

A biblioteca em si é implementada como um conjunto de funções que criam e transformam valores desse tipo, antes de finalmente produzir sua representação como string.

O QuickCheck encoraja uma abordagem em que o desenvolvedor especifica invariantes que devem valer para **quaisquer** dados consumidos pelo código. Para testar a biblioteca, então, precisamos de uma fonte de valores `Doc` aleatórios. Para isso, usamos a pequena suíte de combinadores que o QuickCheck fornece via a classe `Arbitrary`:

```haskell
-- definida em Test.QuickCheck
class Arbitrary a where
  arbitrary :: Gen a
```

Note que os geradores executam em um ambiente `Gen`, indicado pelo tipo. Trata-se de um _monad_ simples de passagem de estado, usado para esconder o estado do gerador de números aleatórios que fica espalhado pelo código. Examinaremos monads minuciosamente nos próximos capítulos; por ora, basta saber que, como `Gen` é um monad, podemos usar a sintaxe `do` para escrever geradores que acessam implicitamente os números aleatórios. Para escrever geradores dos nossos próprios tipos, combinamos as funções que a biblioteca oferece — as principais são:

```haskell
-- definidas em Test.QuickCheck.Gen
elements :: [a] -> Gen a
choose   :: Random a => (a, a) -> Gen a
oneof    :: [Gen a] -> Gen a
```

A função `elements` recebe uma lista de valores e retorna um gerador que escolhe aleatoriamente um deles. Usaremos `choose` e `oneof` em seguida. Com isso, podemos escrever geradores para tipos simples. Para praticar, adicionaremos ao módulo `QuickTestes` um tipo novo, para lógica ternária:

```haskell
-- src/QuickTestes.hs
data Ternary
    = Yes
    | No
    | Unknown
    deriving (Eq, Show)
```

Escrevemos uma instância de `Arbitrary` para `Ternary` escolhendo um elemento da lista dos valores possíveis:

```haskell
-- src/QuickTestes.hs
instance Arbitrary Ternary where
    arbitrary = elements [Yes, No, Unknown]
```

Com isso, já é possível gerar dados aleatórios para o tipo:

```
ghci> :r
ghci> generate arbitrary :: IO [Ternary]
[Unknown,Unknown,No,Yes,Yes,No,Yes,No,Unknown,No,No,Unknown,Yes]
```

Outra abordagem é gerar valores de um tipo básico e **traduzi-los** para o tipo que nos interessa. Poderíamos ter escrito a instância de `Ternary` gerando inteiros de 0 a 2 com `choose` e mapeando-os para os construtores:

```haskell
instance Arbitrary Ternary where
    arbitrary = do
        n <- choose (0, 2) :: Gen Int
        return $ case n of
                      0 -> Yes
                      1 -> No
                      _ -> Unknown
```

Para tipos enumerados, as duas abordagens funcionam bem. Para tipos-produto (como registros e tuplas), geramos cada componente separadamente (e recursivamente, se aninhados) e depois os combinamos. É assim que a própria biblioteca define o gerador de pares:

```haskell
-- definida em Test.QuickCheck.Arbitrary
instance (Arbitrary a, Arbitrary b) => Arbitrary (a, b) where
  arbitrary = do
      x <- arbitrary
      y <- arbitrary
      return (x, y)
```

Por exemplo, gerando tuplas de inteiros:

```
ghci> generate arbitrary :: IO [(Int,Int)]
[(-1,18),(-25,7),(-24,-15),(20,-8),(20,3),(-29,-3),(-19,6),(-13,17)]
```

!!! tip
    **E os caracteres?** Quando o livro foi escrito, o QuickCheck **não tinha** uma instância padrão para `Char` — havia dúvidas sobre qual codificação usar —, e era preciso escrever uma à mão. Nas versões atuais essa instância **já existe** em `Test.QuickCheck`, e é bem mais rica que a gambiarra da época: ela gera todo o espectro Unicode, com viés para caracteres ASCII comuns e caracteres de controle (justamente os que costumam revelar bugs de escape — repare neles nas saídas a seguir). Ou seja: **não defina uma instância de `Char`**; apenas use `arbitrary`.

Vamos então ao gerador para todas as variantes do tipo `Doc`. Quebramos o problema: escolhemos aleatoriamente um construtor e, dependendo dele, geramos seus campos — recursivamente, nos casos de concatenação e união. Escreveremos o código diretamente no `test/Spec.hs`, com um `main` provisório que só imprime alguns documentos gerados:

```haskell
-- test/Spec.hs
import Test.QuickCheck
import Prettify

instance Arbitrary Doc where
    arbitrary = do
        n <- choose (1,6) :: Gen Int
        case n of
             1 -> return Empty

             2 -> do x <- arbitrary
                     return (Char x)

             3 -> do x <- arbitrary
                     return (Text x)

             4 -> return Line

             5 -> do x <- arbitrary
                     y <- arbitrary
                     return (Concat x y)

             6 -> do x <- arbitrary
                     y <- arbitrary
                     return (Union x y)

main :: IO ()
main = do
    docs <- generate arbitrary :: IO [Doc]
    print docs
```

!!! warning
    **Isso ainda não compila — e o erro é instrutivo.** Na Parte 1, exportamos `Doc` como um tipo **abstrato** (`Doc`, sem os construtores) — exatamente para que ninguém de fora pudesse construir ou inspecionar documentos na mão. Mas é **isso** que o nosso gerador e as nossas propriedades precisam fazer! Há uma tensão real aqui entre encapsulamento e testabilidade. A solução mais simples, que adotaremos, é passar a exportar os construtores: no `src/Prettify.hs`, troque `Doc` por `Doc(..)` na lista de exportação. (Em bibliotecas de verdade, o padrão comum é um módulo `*.Internal` que exporta tudo — os testes importam o Internal, e os usuários, a fachada abstrata.)

Feito o ajuste, execute:

```
$ stack test
hs2json> test (suite: hs2json-test)

[Text "o\38884\DC3R\201400?\EOT#;;/\\Gk_y\1061091\178450\&7(4'\174004-A",Text "8\986417\&7",Concat Line (Union Empty (Char '4')),Char '\NUL',Text "\68902\ACKQTA\SOH^Q\200597h\SIh\36934"]
```

_(Saída ilustrativa, encurtada — a sua será diferente.)_ Examinando-a, vemos uma boa mistura: casos básicos, textos cheios de caracteres Unicode e de controle, e documentos aninhados. A cada execução de teste, centenas desses serão gerados.

Essa abordagem foi bem direta, e podemos melhorá-la usando a função `oneof` (cujo tipo vimos acima) para escolher entre geradores de uma lista — e o combinador monádico `liftM` (do módulo `Control.Monad`) para evitar nomear os resultados intermediários:

```haskell
-- test/Spec.hs
instance Arbitrary Doc where
    arbitrary =
        oneof [ return Empty
              , liftM  Char   arbitrary
              , liftM  Text   arbitrary
              , return Line
              , liftM2 Concat arbitrary arbitrary
              , liftM2 Union  arbitrary arbitrary ]
```

Esta versão é mais concisa — apenas escolhe de uma lista de geradores —, mas ambas descrevem os mesmos dados.

### Testando a construção de documentos

Duas das funções básicas sobre documentos são o documento nulo, `empty`, e o operador de concatenação. Revendo suas definições:

```haskell
-- src/Prettify.hs
empty :: Doc
empty = Empty

(<>) :: Doc -> Doc -> Doc
Empty <> y = y
x <> Empty = x
x <> y = x `Concat` y
```

Juntas, elas devem satisfazer uma propriedade razoável: concatenar um documento com o vazio — de qualquer lado — deve deixá-lo inalterado. (É a propriedade de **identidade** que anunciamos no "momento matemático" da Parte 1.) Podemos afirmar a invariante assim:

```haskell
-- test/Spec.hs
prop_empty_id x =
    empty <> x == x
  &&
    x <> empty == x
```

E confirmar que ela vale, direto no GHCi (`stack ghci --test` carrega também o componente de testes):

```
ghci> quickCheck prop_empty_id
+++ OK, passed 100 tests.
```

_(Repare que aqui não precisamos de anotação de tipo: o `<>` do Prettify força `x :: Doc`, e a nossa instância `Arbitrary Doc` faz o resto.)_

### Executando tudo com o `stack test`: `quickCheckAll`

_(Esta seção foi reescrita nesta edição.)_

Rodar `quickCheck` propriedade por propriedade no GHCi é ótimo para explorar, mas queremos que o **`stack test`** execute todas de uma vez. O QuickCheck traz um utilitário para isso: `quickCheckAll`, que usa **Template Haskell** — o mecanismo de metaprogramação do GHC — para localizar, em tempo de compilação, todas as funções do módulo cujo nome começa com `prop_`, e gerar o código que as executa.

Três detalhes fazem tudo funcionar:

1. O pragma `{-# LANGUAGE TemplateHaskell #-}` no topo do arquivo, habilitando a extensão;
2. A linha `return []` antes da definição — um truque necessário para que o Template Haskell "enxergue" todas as definições que vieram acima dela no arquivo;
3. A invocação `$quickCheckAll` (o `$` executa a metafunção em tempo de compilação), que produz uma ação `IO Bool`: `True` se todas as propriedades passaram.

Há ainda um detalhe que o livro original deixou passar, e que vale corrigir: **o processo de testes precisa terminar com código de saída de erro quando algo falha**. É só o código de saída que o `stack test` (e qualquer ferramenta de integração contínua) olha — sem isso, a suíte imprime "falhou" mas o Stack alegremente reporta `Test suite passed`. Resolvemos com `exitFailure`, do módulo `System.Exit`.

O `test/Spec.hs` completo até aqui:

```haskell
-- test/Spec.hs
{-# LANGUAGE TemplateHaskell #-}

import Prelude hiding ((<>))

import Test.QuickCheck
import Data.List (intersperse)
import Control.Monad (liftM, liftM2)
import System.Exit (exitFailure)

import Prettify

instance Arbitrary Doc where
    arbitrary =
        oneof [ return Empty
              , liftM  Char   arbitrary
              , liftM  Text   arbitrary
              , return Line
              , liftM2 Concat arbitrary arbitrary
              , liftM2 Union  arbitrary arbitrary ]

prop_empty_id x =
    empty <> x == x
  &&
    x <> empty == x

return []
runTests = $quickCheckAll

main :: IO ()
main = do
    passed <- runTests
    if passed
        then putStrLn "Passou em todos os testes."
        else do putStrLn "Alguns testes falharam."
                exitFailure
```

_(Note o `import Prelude hiding ((<>))` — o mesmo ajuste dos módulos da Parte 1, pois usamos aqui o `<>` do Prettify, não o do Prelude.)_

Executando:

```
$ stack test
hs2json> test (suite: hs2json-test)

=== prop_empty_id from test/Spec.hs:22 ===
+++ OK, passed 100 tests.

Passou em todos os testes.

hs2json> Test suite hs2json-test passed
```

Outras funções da API são simples o suficiente para terem o comportamento **completamente** descrito por propriedades. Revendo suas definições no `Prettify`:

```haskell
-- src/Prettify.hs
char :: Char -> Doc
char c = Char c

text :: String -> Doc
text "" = Empty
text s  = Text s

double :: Double -> Doc
double d = text (show d)

line :: Doc
line = Line
```

Escrevemos, então, os testes correspondentes — assim, modificações futuras não quebrarão estas invariantes básicas sem que a suíte grite:

```haskell
-- test/Spec.hs
prop_char c   = char c   == Char c
prop_text s   = text s   == if null s then Empty else Text s
prop_line     = line     == Line
prop_double d = double d == text (show d)
```

Essas propriedades bastam para testar completamente a estrutura retornada pelos operadores básicos de documentos.

### Usando listas como modelos

Funções de alta ordem são a base de programas reutilizáveis, e nossa biblioteca não é exceção: um `fold` customizado é usado internamente para implementar tanto a concatenação quanto a intercalação de separadores:

```haskell
-- src/Prettify.hs
fold :: (Doc -> Doc -> Doc) -> [Doc] -> Doc
fold f = foldr f empty

hcat :: [Doc] -> Doc
hcat = fold (<>)
```

Podemos testar instâncias específicas do `fold` isoladamente. A concatenação horizontal, por exemplo, é fácil de especificar escrevendo uma implementação de referência sobre listas:

```haskell
-- test/Spec.hs
prop_hcat xs = hcat xs == glue xs
    where
        glue []     = empty
        glue (d:ds) = d <> glue ds
```

História parecida com `punctuate`, cuja inserção de pontuação parece se modelar com a intercalação de listas (`intersperse`, de `Data.List`, recebe um elemento e o intercala entre os elementos de uma lista):

```haskell
-- test/Spec.hs
prop_punctuate s xs = punctuate s xs == intersperse s xs
```

Embora pareça correta, a execução revela uma falha na nossa lógica:

```
$ stack test

=== prop_punctuate from test/Spec.hs:37 ===
*** Failed! Falsified (after 6 tests):
Char '}'
[Text "\DC3v\DEL~w",Concat Empty (Text "\199132\NAK")]

Alguns testes falharam.
```

_(O QuickCheck moderno diz `Falsified` onde o antigo dizia `Falsifiable`. As duas linhas após o cabeçalho são os argumentos do contraexemplo: o separador `s` e a lista `xs` — repare no `Empty` dentro de um `Concat`, a pista do problema. E, graças ao nosso `exitFailure`, desta vez o `stack test` termina, corretamente, reportando a falha da suíte.)_

A biblioteca **otimiza fora os documentos vazios redundantes** (lembre-se dos casos `Empty` do `<>`), algo que o modelo de lista não faz — então precisamos enriquecer o modelo para casar com a realidade. Primeiro intercalamos a pontuação e depois eliminamos os `Empty` espalhados, assim:

```haskell
-- test/Spec.hs
prop_punctuate' s xs = punctuate s xs == combine (intersperse s xs)
    where
        combine []           = []
        combine [x]          = [x]
        combine (x:Empty:ys) = x : combine ys
        combine (Empty:y:ys) = y : combine ys
        combine (x:y:ys)     = x `Concat` y : combine ys
```

Executando (e removendo a versão ingênua, `prop_punctuate`), confirmamos o resultado. É reconfortante que o framework localize falhas na lógica que expressamos — é exatamente para isso que ele existe:

```
$ stack test

=== prop_empty_id from test/Spec.hs:22 ===
+++ OK, passed 100 tests.

=== prop_char from test/Spec.hs:27 ===
+++ OK, passed 100 tests.

=== prop_text from test/Spec.hs:28 ===
+++ OK, passed 100 tests.

=== prop_line from test/Spec.hs:29 ===
+++ OK, passed 1 test.

=== prop_double from test/Spec.hs:30 ===
+++ OK, passed 100 tests.

=== prop_hcat from test/Spec.hs:32 ===
+++ OK, passed 100 tests.

=== prop_punctuate' from test/Spec.hs:37 ===
+++ OK, passed 100 tests.

Passou em todos os testes.

hs2json> Test suite hs2json-test passed
```

_(Curiosidade: `prop_line` não recebe argumentos, então não há o que gerar — o QuickCheck a trata como um teste único: `passed 1 test`.)_

!!! tip
    **Sobre o encolhimento (shrinking):** quando uma propriedade falha, o QuickCheck tenta **encolher** o contraexemplo — remover elementos, diminuir números — até achar o menor caso que ainda falha, o que facilita muito a depuração (você verá mensagens como `Failed! ... and 3 shrinks`). Isso funciona automaticamente para os tipos embutidos; para o nosso `Doc`, não definimos o método `shrink` da classe `Arbitrary`, então os contraexemplos vêm "crus". Implementá-lo (dica: `genericShrink`) fica como exercício.

