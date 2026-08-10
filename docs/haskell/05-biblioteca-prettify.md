# Construindo a Biblioteca Prettify

## Criando a biblioteca de impressão agradável

No módulo `Prettify`, representamos o tipo `Doc` como um tipo de dados algébrico:

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

Observe que o tipo `Doc` é, na verdade, uma **árvore**. Os construtores `Concat` e `Union` criam um nó interno a partir de outros dois valores `Doc`, enquanto `Empty` e os demais construtores simples formam as folhas.

No cabeçalho do módulo, exportaremos o nome do tipo, **mas não seus construtores** (repare: `Doc`, e não `Doc(..)`). Isso impedirá que módulos que usem `Doc` criem valores diretamente ou casem padrões com eles — é o tipo abstrato de que falamos:

```haskell
-- src/Prettify.hs
module Prettify
    (
      Doc
    , empty
    , char
    , text
    , double
    , line
    , (<>)
    , hcat
    , fsep
    , (</>)
    , punctuate
    , group
    , softline
    , compact
    , pretty
    ) where

import Prelude hiding ((<>))
```

Em vez de criar um `Doc` na mão, um usuário do módulo `Prettify` chamará uma função que fornecemos. Eis as funções de construção simples. À medida que adicionamos as definições reais, **substituímos os esboços** que estavam no `Prettify.hs`:

```haskell
-- src/Prettify.hs
empty :: Doc
empty = Empty

char :: Char -> Doc
char c = Char c

text :: String -> Doc
text "" = Empty
text s  = Text s

double :: Double -> Doc
double d = text (show d)
```

O construtor `Line` representa uma quebra de linha. A função `line` cria uma quebra de linha _hard_, que sempre aparece na saída da biblioteca. Às vezes queremos uma quebra de linha _soft_, usada somente se a linha for grande demais para caber na janela ou página — introduziremos a função `softline` em breve.

```haskell
-- src/Prettify.hs
line :: Doc
line = Line
```

Quase tão simples quanto os construtores básicos é a função `(<>)`, que concatena dois valores `Doc`:

```haskell
-- src/Prettify.hs
(<>) :: Doc -> Doc -> Doc
Empty <> y = y
x <> Empty = x
x <> y = x `Concat` y
```

Casamos o padrão `Empty` de forma que concatenar um `Doc` com `Empty` à esquerda ou à direita não tenha efeito. Isso evita acrescentar valores inúteis à árvore:

```
ghci> text "foo" <> text "bar"
Concat (Text "foo") (Text "bar")
ghci> text "foo" <> empty
Text "foo"
ghci> empty <> text "bar"
Text "bar"
```

!!! note
    **Um momento matemático:** se colocarmos brevemente nossos chapéus de matemáticos, podemos dizer que `Empty` é a **identidade** da concatenação, pois nada acontece ao concatenar um `Doc` com `Empty` — assim como `0` é a identidade da adição e `1` a da multiplicação. Essa perspectiva tem consequências muito úteis. Aliás, aqui está a piada interna do Haskell moderno: um tipo com uma operação associativa como `<>` é um **`Semigroup`**, e com um elemento identidade como `empty` é um **`Monoid`** — exatamente as classes que hoje vivem no Prelude e que nos obrigaram ao `hiding ((<>))`. Nosso `Doc` poderia declarar instâncias delas; fica como exploração para depois das classes de tipos (Capítulo 6).

Nossas funções `hcat` e `fsep` concatenam uma lista de `Doc` em um só. Lembre-se de que podemos definir a concatenação de listas usando `foldr`:

```haskell
concat :: [[a]] -> [a]
concat = foldr (++) []
```

Como `(<>)` é análogo a `(++)`, e `empty` a `[]`, podemos escrever `hcat` e `fsep` como _folds_ também:

```haskell
-- src/Prettify.hs
hcat :: [Doc] -> Doc
hcat = fold (<>)

fold :: (Doc -> Doc -> Doc) -> [Doc] -> Doc
fold f = foldr f empty
```

A definição de `fsep` depende de várias outras funções:

```haskell
-- src/Prettify.hs
fsep :: [Doc] -> Doc
fsep = fold (</>)

(</>) :: Doc -> Doc -> Doc
x </> y = x <> softline <> y

softline :: Doc
softline = group line
```

Isso merece uma explicação. A `softline` deve inserir uma nova linha se a linha atual ficar muito grande, ou um espaço, caso contrário. Como fazer isso, se o tipo `Doc` não sabe nada sobre renderização? Nossa resposta: toda vez que encontramos uma linha soft, mantemos **duas representações alternativas** do documento, usando o construtor `Union`:

```haskell
-- src/Prettify.hs
group :: Doc -> Doc
group x = flatten x `Union` x
```

Nossa função `flatten` substitui cada `Line` por um espaço, transformando duas linhas em uma só:

```haskell
-- src/Prettify.hs
flatten :: Doc -> Doc
flatten (x `Concat` y) = flatten x `Concat` flatten y
flatten Line           = Char ' '
flatten (x `Union` _)  = flatten x
flatten other          = other
```

Note que sempre chamamos `flatten` no lado **esquerdo** de uma `Union`: esse lado tem sempre o mesmo tamanho (em caracteres) ou é maior que o direito. Usaremos essa propriedade na função de renderização adiante.

### Renderização compacta

Frequentemente precisamos da representação de uma informação com o mínimo de caracteres possível. Se estamos enviando JSON por uma conexão de rede, não faz sentido deixá-lo "bonito": o software do outro lado não se importa, e os espaços em branco do layout só adicionam sobrecarga.

Para esses casos, e por ser um pedaço de código simples para começar, forneceremos a função `compact`:

```haskell
-- src/Prettify.hs
compact :: Doc -> String
compact x = transform [x]
    where transform [] = ""
          transform (d:ds) =
              case d of
                Empty        -> transform ds
                Char c       -> c : transform ds
                Text s       -> s ++ transform ds
                Line         -> '\n' : transform ds
                a `Concat` b -> transform (a:b:ds)
                _ `Union` b  -> transform (b:ds)
```

A função `compact` envolve seu argumento em uma lista e aplica a auxiliar `transform`, que trata o argumento como uma **pilha** de itens a processar, onde o primeiro elemento da lista é o topo.

A `transform` usa o padrão `(d:ds)` para quebrar a pilha em topo, `d`, e restante, `ds`. Na expressão `case`, os primeiros ramos fazem recursão sobre `ds`, consumindo um item da pilha por chamada. Os dois últimos ramos **adicionam** itens à frente de `ds`: o ramo `Concat` adiciona ambos os elementos à pilha, enquanto o ramo `Union` ignora o elemento esquerdo (aquele em que chamamos `flatten`) e adiciona o direito.

Agora já preenchemos definições suficientes para experimentar a `compact` no GHCi:

```
$ stack ghci
ghci> import Prettify
ghci> import PrettyJSON
ghci> let value = renderJValue (JObject [("f", JNumber 1), ("q", JBool True)])
ghci> :type value
value :: Doc
ghci> putStrLn (compact value)
{"f": 1.0,
"q": true
}
```

Para entender melhor como o código funciona, olhemos um exemplo mais simples em detalhe:

```
ghci> char 'f' <> text "oo"
Concat (Char 'f') (Text "oo")
ghci> compact (char 'f' <> text "oo")
"foo"
```

1. Quando aplicamos `compact`, ela põe o argumento numa lista e aplica `transform`.
2. A `transform` recebe uma lista de um item, que casa com `(d:ds)`. Então `d` é `Concat (Char 'f') (Text "oo")` e `ds` é a lista vazia, `[]`.
3. Como o construtor de `d` é `Concat`, o padrão `Concat` casa na expressão `case`. No lado direito, adicionamos `Char 'f'` e `Text "oo"` à pilha e aplicamos `transform` recursivamente.
4. A `transform` recebe uma lista de dois itens, casando de novo com `(d:ds)`. Agora `d` é `Char 'f'` e `ds` é `[Text "oo"]`.
5. O `case` casa no ramo `Char`. No lado direito, usamos `(:)` para construir uma lista cuja cabeça é `'f'` e cujo restante é a aplicação recursiva de `transform`.
6. A chamada recursiva recebe um item: `d` é `Text "oo"`, e `ds` é `[]`.
7. O `case` casa no ramo `Text`. Usamos `(++)` para concatenar `"oo"` com o resultado da chamada recursiva.
8. Na invocação final, `transform` recebe a lista vazia e retorna a string vazia.
9. O resultado é `"oo" ++ ""`... e, subindo, `'f' : ("oo" ++ "")` — ou seja, `"foo"`.

### A verdadeira impressão agradável

Enquanto a `compact` é útil para comunicação máquina-a-máquina, seu resultado nem sempre é fácil de um humano acompanhar: há pouquíssima informação em cada linha. Para saídas mais agradáveis, escreveremos outra função, `pretty`. Comparada à `compact`, a `pretty` recebe um argumento a mais: a largura máxima da linha, em colunas. (Assumimos uma fonte de largura fixa.)

```haskell
-- src/Prettify.hs
pretty :: Int -> Doc -> String
```

Para sermos precisos: o parâmetro `Int` controla o comportamento de `pretty` quando ela encontra uma `softline`. Só ali ela tem a opção de continuar na linha atual ou começar uma nova. Nos demais lugares, seguimos rigorosamente as diretrizes estabelecidas por quem construiu o documento.

Eis o núcleo da implementação:

```haskell
-- src/Prettify.hs
pretty width x = best 0 [x]
    where best col (d:ds) =
              case d of
                Empty        -> best col ds
                Char c       -> c :  best (col + 1) ds
                Text s       -> s ++ best (col + length s) ds
                Line         -> '\n' : best 0 ds
                a `Concat` b -> best col (a:b:ds)
                a `Union` b  -> nicest col (best col (a:ds))
                                           (best col (b:ds))
          best _ _ = ""

          nicest col a b | (width - least) `fits` a = a
                         | otherwise                = b
                         where least = min width col
```

Nossa auxiliar `best` recebe dois argumentos: o número de colunas já usadas na linha atual e a lista dos valores `Doc` que restam processar.

Nos casos simples, `best` atualiza a variável `col` de maneira direta conforme consome a entrada. Até o caso `Concat` é óbvio: empilhamos os dois componentes e não tocamos em `col`.

O caso interessante é o construtor `Union`. Lembre que aplicamos `flatten` ao elemento da esquerda e nada ao da direita; e que `flatten` troca quebras de linha por espaços. Portanto, nosso trabalho é ver **qual dos dois layouts** — o achatado ou o original — cabe na restrição de largura.

Para isso, escrevemos uma pequena auxiliar que determina se uma linha de um valor `Doc` renderizado cabe no número dado de colunas:

```haskell
-- src/Prettify.hs
fits :: Int -> String -> Bool
w `fits` _ | w < 0 = False
w `fits` ""        = True
w `fits` ('\n':_)  = True
w `fits` (c:cs)    = (w - 1) `fits` cs
```

!!! tip
    **Sobre os avisos do compilador:** se você compila com `stack build`, o template do projeto ativa `-Wall`, e o GHC emitirá alguns _warnings_ neste código — por exemplo, `Defined but not used: 'p'` na primeira equação de `punctuate` e avisos similares em `fits`. **Warnings não são erros**: o programa compila e funciona. Eles apontam variáveis nomeadas que não usamos; a convenção idiomática é prefixá-las com sublinhado (`_p`, `_w`) para dizer ao compilador "eu sei, é de propósito". Mantivemos o código como no livro original; silenciar os avisos fica como micro-exercício.

### Seguindo o fluxo de execução

Para entender como esse código funciona, consideremos um valor `Doc` simples:

```
ghci> empty </> char 'a'
Concat (Union (Char ' ') Line) (Char 'a')
```

Vamos aplicar `pretty 2` a esse valor. Na primeira aplicação de `best`, o valor de `col` é zero. O `case` casa com `Concat`, empilha `Union (Char ' ') Line` e `Char 'a'`, e recorre. Na chamada recursiva, casa com `Union (Char ' ') Line`.

Neste ponto, ignoraremos a ordem usual de avaliação do Haskell — isso simplifica a explicação sem mudar o resultado. Temos agora duas subexpressões: `best 0 [Char ' ', Char 'a']` e `best 0 [Line, Char 'a']`. A primeira avalia para `" a"`, e a segunda para `"\na"`. Substituindo na expressão externa, obtemos `nicest 0 " a" "\na"`.

Para entender o resultado de `nicest` aqui, fazemos uma pequena substituição: os valores de `width` e `col` são `2` e `0`, então `least` é `0` e `width - least` é `2`. Avaliamos rapidamente ``2 `fits` " a"`` no GHCi:

```
ghci> 2 `fits` " a"
True
```

Como isso avalia para `True`, o resultado de `nicest` é `" a"`.

Se aplicarmos nossa função `pretty` ao mesmo JSON de antes, veremos que ela produz resultados diferentes dependendo da largura que dermos:

```
ghci> putStrLn (pretty 10 value)
{"f": 1.0,
"q": true
}
ghci> putStrLn (pretty 20 value)
{"f": 1.0, "q": true
}
ghci> putStrLn (pretty 30 value)
{"f": 1.0, "q": true }
```

