# Preparando o Projeto e QuickCheck Básico

*Parte 2 — Testes e garantia de qualidade*

Construir sistemas reais significa ter cuidado com controle de qualidade, robustez e corretude. Com os mecanismos certos de garantia de qualidade, código bem escrito pode parecer uma máquina precisa, com todas as funções executando suas tarefas de acordo com as especificações. Não há desleixo nas situações críticas, e o resultado é código autoexplicativo — e obviamente correto — do tipo que inspira confiança.

Em Haskell, existem diversas ferramentas à disposição para construir sistemas assim. A mais óbvia, embutida na própria linguagem, é o sistema de tipos expressivo, que permite impor restrições verificadas estaticamente — tornando impossível escrever código que as viole. Adicionalmente, pureza e polimorfismo promovem um estilo de código modular, refatorável e testável.

Os testes têm papel central em manter o código no caminho certo. Os principais mecanismos de teste em Haskell são o tradicional teste de unidade (por meio da biblioteca HUnit) e seu descendente mais poderoso: o **teste baseado em propriedades**, através do **QuickCheck**, um framework de testes de código aberto. Testes baseados em propriedades promovem uma abordagem de alto nível, na forma de **invariantes** que as funções devem satisfazer universalmente, com os dados de teste **gerados pela biblioteca** para o programador. Assim, o código pode ser martelado com milhares de testes que seriam inviáveis de escrever à mão, cobrindo casos-limite que dificilmente encontraríamos de outra forma.

Neste capítulo, veremos como usar o QuickCheck para estabelecer invariantes no código e então reexaminaremos o _pretty printer_ desenvolvido na Parte 1, testando-o com o framework. Também veremos como acompanhar o processo com a ferramenta de cobertura de testes do GHC: o HPC.

## Preparando o projeto

Se você acabou de concluir a Parte 1, já tem tudo pronto — basta continuar no diretório `hs2json`. Caso contrário, clone a biblioteca:

```bash
git clone https://github.com/profsergiocosta/hs2json.git
cd hs2json
```

Agora, execute a suíte de testes:

```
$ stack test
```

Após a compilação, aparecerá a informação de que ainda não existem testes implementados:

```
hs2json> test (suite: hs2json-test)

Test suite not yet implemented

hs2json> Test suite hs2json-test passed
```

Podemos confirmar no código-fonte que nenhum teste foi implementado — este é o `test/Spec.hs` que o template do Stack gerou:

```haskell
-- test/Spec.hs
main :: IO ()
main = putStrLn "Test suite not yet implemented"
```

O objetivo deste capítulo é implementar esses testes.

!!! tip
    **Nota do professor:** a implementação final deste capítulo fica guardada em um repositório separado (`hs2json-test`), para que vocês possam usá-lo como referência se algo der errado. Porém, é importante que **executem os passos a seguir** a partir do repositório _sem_ os testes — o aprendizado está no caminho, não no destino.

### Adicionando o QuickCheck às dependências

O QuickCheck não faz parte do `base`, então precisamos declará-lo no `package.yaml`:

```yaml
# package.yaml
dependencies:
  - base >= 4.7 && < 5
  - QuickCheck
```

No próximo `stack build` (ou `stack test`, ou `stack ghci`), o Stack baixa e compila o QuickCheck automaticamente — é a mágica das dependências explícitas que discutimos na Parte 1.

!!! tip
    Colocado nessa posição, o QuickCheck fica disponível para **todos** os componentes (biblioteca, executável e testes) — é o que queremos aqui, pois escreveremos propriedades também em `src/`. Em projetos reais, dependências usadas _só_ nos testes costumam ser declaradas apenas no componente de testes (dentro de `tests:` no `package.yaml`), para não "vazar" para quem usa a biblioteca.

## QuickCheck: teste baseado em propriedades

Para ter uma ideia de como funcionam os testes baseados em propriedades, começaremos com um cenário simples: você escreveu uma função de ordenação e quer testar seu comportamento. Crie um novo módulo, `src/QuickTestes.hs`, já com as importações que usaremos ao longo da seção:

```haskell
-- src/QuickTestes.hs
module QuickTestes where

import Data.List (sort, (\\))
import Test.QuickCheck
```

E a função que queremos testar — uma rotina personalizada de ordenação:

```haskell
-- src/QuickTestes.hs
qsort :: Ord a => [a] -> [a]
qsort []     = []
qsort (x:xs) = qsort lhs ++ [x] ++ qsort rhs
    where lhs = filter  (< x) xs
          rhs = filter (>= x) xs
```

Esta é a clássica implementação de ordenação em Haskell: um estudo sobre elegância em programação funcional, não sobre eficiência (não é um algoritmo _in-place_). Queremos checar se essa função obedece às regras básicas que uma boa ordenação deve seguir.

Uma invariante útil para começar — e que aparece com frequência em código puramente funcional — é a **idempotência**: aplicar a função duas vezes deve dar o mesmo resultado que aplicá-la uma vez. Para uma rotina de ordenação, isso deve valer sempre, ou a situação vai ficar feia. A invariante pode ser codificada como uma simples propriedade:

```haskell
-- src/QuickTestes.hs
prop_idempotent xs = qsort (qsort xs) == qsort xs
```

Usaremos a convenção do QuickCheck de prefixar as propriedades com `prop_`, para diferenciá-las do código normal. A propriedade de idempotência é só uma função Haskell declarando uma igualdade que deve valer para qualquer entrada. Podemos checar manualmente que ela faz sentido para alguns casos:

```
$ stack ghci
ghci> prop_idempotent []
True
ghci> prop_idempotent [1,1,1,1]
True
ghci> prop_idempotent [1..100]
True
ghci> prop_idempotent [1,5,2,1,2,0,9]
True
```

Parece certo. Entretanto, escrever entradas à mão é tedioso e viola o código moral do programador funcional eficiente: **deixe a máquina fazer o trabalho!** Para automatizar isso, o QuickCheck traz geradores de dados para todos os tipos básicos do Haskell, usando a _typeclass_ `Arbitrary` como interface uniforme para a geração pseudoaleatória — e o sistema de tipos para decidir qual gerador usar. O QuickCheck normalmente esconde a geração de dados, mas podemos executar os geradores à mão para espiar o que ele produz. Por exemplo, gerando uma lista aleatória de booleanos:

```
ghci> import Test.QuickCheck
ghci> generate arbitrary :: IO [Bool]
[True,True,True,True,False,True,False,True,False,False,True,True]
```

_(A saída varia a cada execução, claro — os dados são aleatórios.)_

O QuickCheck gera dados assim e os passa à propriedade da nossa escolha, por meio da função `quickCheck`. O tipo da propriedade determina qual gerador é usado; o `quickCheck` então verifica que a propriedade vale para todos os dados produzidos. Como nossa propriedade é **polimórfica** na lista, precisamos escolher um tipo concreto para o qual gerar os dados, o que escrevemos como uma restrição de tipo (caso contrário, o GHCi escolheria o desinteressante tipo `()` para os elementos):

```
ghci> quickCheck (prop_idempotent :: [Integer] -> Bool)
+++ OK, passed 100 tests.
```

Para 100 listas diferentes geradas, a propriedade foi satisfeita. Ao escrever testes, costuma ser útil ver os dados gerados em cada caso. Para isso, trocamos `quickCheck` pelo seu irmão verboso, `verboseCheck`. Mostrando só o começo da saída:

```
ghci> verboseCheck (prop_idempotent :: [Integer] -> Bool)
Passed:
[]

Passed:
[1]

Passed:
[-2,-2,3]

Passed:
[-2,0,2]

...
+++ OK, passed 100 tests.
```

Observe que os testes são aplicados a listas de tamanhos variados (o QuickCheck começa com entradas pequenas e vai crescendo). Agora, vamos a propriedades mais sofisticadas.

### Testes de propriedade

Boas bibliotecas consistem em um conjunto de primitivas ortogonais com relações sensatas entre si. Podemos usar o QuickCheck para **especificar essas relações**, o que nos ajuda inclusive a desenhar uma boa interface: o QuickCheck age como uma ferramenta de "lint" da consistência da biblioteca.

Nossa função de ordenação certamente se relaciona com outras operações de lista. Por exemplo: o primeiro elemento de uma lista ordenada deve ser o **menor** elemento da entrada. Ficamos tentados a expressar essa intuição usando a função `minimum`:

```haskell
-- src/QuickTestes.hs
prop_minimum xs = head (qsort xs) == minimum xs
```

Ao recarregar o módulo (`:r`) e testar a nova propriedade, encontraremos um erro:

```
ghci> :r
ghci> quickCheck (prop_minimum :: [Integer] -> Bool)
*** Failed! (after 1 test):
Exception:
  Prelude.head: empty list
  ...
  head, called at src/QuickTestes.hs:15:19 in main:QuickTestes
[]
```

*(No livro original a mensagem era uma linha só; o QuickCheck moderno mostra a exceção com o *call stack*, apontando a linha exata do `head` culpado — bem mais útil. A última linha, `[]`, é o **contraexemplo**: a entrada que quebrou a propriedade.)*

A propriedade falhou ao ordenar a lista **vazia** — para a qual `head` e `minimum` não estão definidas, como vemos nas suas definições:

```haskell
-- definidas no Prelude
head :: [a] -> a
head (x:_) = x
head []    = error "Prelude.head: empty list"

minimum :: (Ord a) => [a] -> a
minimum [] = error "Prelude.minimum: empty list"
minimum xs = foldl1 min xs
```

Portanto, a propriedade só faz sentido para listas não vazias. Felizmente, o QuickCheck vem com uma pequena linguagem de propriedades que nos permite ser mais precisos sobre as invariantes, descartando entradas que não queremos considerar. Para o caso da lista vazia, o que queremos dizer é: **se** a lista não é vazia, **então** o primeiro elemento da ordenação é o mínimo. A função de implicação `(==>)` descarta os dados inválidos antes de testar:

```haskell
-- src/QuickTestes.hs
prop_minimum' xs = not (null xs) ==> head (qsort xs) == minimum xs
```

Removido o caso da lista vazia, confirmamos que a propriedade de fato vale:

```
ghci> quickCheck (prop_minimum' :: [Integer] -> Property)
+++ OK, passed 100 tests; 14 discarded.
```

_(O número de descartados — as listas vazias geradas e ignoradas — varia a cada execução.)_

Note que tivemos que mudar o **tipo** da propriedade: de um simples `Bool` para o tipo mais geral `Property` (a propriedade agora é um valor que filtra as entradas antes de testar, e não uma constante booleana).

Podemos completar o conjunto básico com outras invariantes que a ordenação deve satisfazer: a saída deve estar **ordenada** (cada elemento menor ou igual ao sucessor); a saída deve ser uma **permutação** da entrada (via a diferença de listas, `(\\)`); o último elemento deve ser o **máximo**; e o mínimo de duas listas concatenadas e ordenadas deve ser o menor dos mínimos delas:

```haskell
-- src/QuickTestes.hs
prop_ordered xs = ordered (qsort xs)
    where ordered []       = True
          ordered [x]      = True
          ordered (x:y:ys) = x <= y && ordered (y:ys)

prop_permutation xs = permutation xs (qsort xs)
    where permutation as bs = null (as \\ bs) && null (bs \\ as)

prop_maximum xs =
    not (null xs) ==>
        last (qsort xs) == maximum xs

prop_append xs ys =
    not (null xs) ==>
    not (null ys) ==>
        head (qsort (xs ++ ys)) == min (minimum xs) (minimum ys)
```

### Testando sobre um modelo

Outra técnica para ganhar confiança no código é testar contra uma **implementação-modelo**. Podemos relacionar a nossa ordenação com a função `sort` da biblioteca padrão: se elas se comportam igual, ganhamos confiança de que a nossa faz a coisa certa.

```haskell
-- src/QuickTestes.hs
prop_sort_model xs = sort xs == qsort xs
```

```
ghci> quickCheck (prop_sort_model :: [Integer] -> Bool)
+++ OK, passed 100 tests.
```

Esse tipo de teste baseado em modelo é extremamente poderoso. Frequentemente, desenvolvedores têm uma implementação de referência ou protótipo que, embora ineficiente, é correta. Ela pode ser mantida por perto para assegurar que o código de produção otimizado continua de acordo com a referência. Construindo uma grande suíte desses testes e executando-a regularmente (a cada commit, por exemplo), garantimos baratíssimo a precisão do código. Grandes projetos Haskell costumam ter suítes de propriedades de tamanho comparável ao do próprio projeto, com milhares de invariantes testadas a cada mudança.

