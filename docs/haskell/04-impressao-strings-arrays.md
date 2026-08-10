# Impressão Agradável de Strings, Arrays e Objetos

## Impressão agradável de uma string

Quando precisamos imprimir uma string, o JSON impõe regras de escape moderadamente complexas que devemos seguir. No nível mais alto, uma string é apenas uma série de caracteres entre aspas.

Estas funções fazem parte do **renderizador**, então vão em `PrettyJSON.hs`:

```haskell
-- src/PrettyJSON.hs
string :: String -> Doc
string = enclose '"' '"' . hcat . map oneChar
```

!!! note
    **Estilo ponto-livre:** este estilo de escrever uma definição exclusivamente como composição de outras funções é chamado de _estilo ponto-livre_ (point-free). O uso da palavra "ponto" **não** se refere ao caractere `.` da composição; o termo é aproximadamente sinônimo (em Haskell) de _valor_ — uma definição ponto-livre não menciona o valor sobre o qual opera.

    Compare a definição ponto-livre de `string`, acima, com esta versão "pointy", que usa a variável `s` para se referir ao valor:

    ```haskell
    pointyString :: String -> Doc
    pointyString s = enclose '"' '"' (hcat (map oneChar s))
    ```

A função `enclose` simplesmente põe um valor `Doc` entre um caractere de abertura e um de fechamento:

```haskell
-- src/PrettyJSON.hs
enclose :: Char -> Char -> Doc -> Doc
enclose left right x = char left <> x <> char right
```

O operador `(<>)` será fornecido pela nossa biblioteca `Prettify`. Ele concatena dois valores `Doc` — é o análogo, para documentos, do `(++)` das listas. Adicione os esboços ao `Prettify.hs`:

```haskell
-- src/Prettify.hs
(<>) :: Doc -> Doc -> Doc
a <> b = undefined

char :: Char -> Doc
char c = undefined
```

_(Lembre: o `import Prelude hiding ((<>))` no topo do `Prettify.hs` é o que nos permite definir nosso próprio `<>` sem ambiguidade.)_

Nossa biblioteca `Prettify` também fornece `hcat`, que concatena múltiplos valores `Doc` em um só — análogo ao `concat` para listas:

```haskell
-- src/Prettify.hs
hcat :: [Doc] -> Doc
hcat xs = undefined
```

Nossa função `string` aplica `oneChar` a cada caractere da string, concatena tudo, e põe o resultado entre aspas. A função `oneChar` escapa ou renderiza um caractere individual:

```haskell
-- src/PrettyJSON.hs
oneChar :: Char -> Doc
oneChar c = case lookup c simpleEscapes of
              Just r -> text r
              Nothing | mustEscape c -> hexEscape c
                      | otherwise    -> char c
    where mustEscape ch = ch < ' ' || ch == '\x7f' || ch > '\xff'

simpleEscapes :: [(Char, String)]
simpleEscapes = zipWith ch "\b\n\f\r\t\\\"/" "bnfrt\\\"/"
    where ch a b = (a, ['\\',b])
```

O valor `simpleEscapes` é uma lista de pares. Chamamos uma lista de pares de _lista de associação_, ou simplesmente _alist_ (de _association list_). Cada elemento da nossa alist associa um caractere à sua versão escapada:

```
ghci> take 4 simpleEscapes
[('\b',"\\b"),('\n',"\\n"),('\f',"\\f"),('\r',"\\r")]
```

Nossa expressão `case` tenta casar o caractere com a alist. Se encontramos uma correspondência, a emitimos; caso contrário, talvez precisemos escapar o caractere de uma forma mais complicada, e nesse caso realizamos esse escape. Somente se nenhum escape é necessário emitimos o caractere como texto puro. Para sermos conservadores, os únicos caracteres sem escape que emitimos são os ASCII imprimíveis.

O escape mais sofisticado envolve transformar o caractere na string `"\u"` seguida de uma sequência de quatro caracteres hexadecimais representando o valor numérico do caractere Unicode:

```haskell
-- src/PrettyJSON.hs
smallHex :: Int -> Doc
smallHex x  = text "\\u"
           <> text (replicate (4 - length h) '0')
           <> text h
    where h = showHex x ""
```

A função `showHex` vem do módulo `Numeric` (você precisará importá-lo no início do `PrettyJSON.hs`) e retorna a representação hexadecimal de um número:

```
ghci> import Numeric
ghci> showHex 114111 ""
"1bdbf"
```

A função `replicate` é fornecida pelo Prelude e cria uma lista de tamanho fixo com o elemento repetido:

```
ghci> replicate 5 "foo"
["foo","foo","foo","foo","foo"]
```

Há um problema: a codificação de quatro dígitos do `smallHex` só consegue representar caracteres Unicode até `0xffff`, mas caracteres Unicode válidos vão até `0x10ffff`. Para representar corretamente um caractere acima de `0xffff` em uma string JSON, seguimos regras (complicadas) que o dividem em **dois** valores de 16 bits — os chamados _pares substitutos_ (surrogate pairs). Isso nos dá a oportunidade de fazer manipulação de bits em Haskell:

```haskell
-- src/PrettyJSON.hs
astral :: Int -> Doc
astral n = smallHex (a + 0xd800) <> smallHex (b + 0xdc00)
    where a = (n `shiftR` 10) .&. 0x3ff
          b = n .&. 0x3ff
```

A função `shiftR`, do módulo `Data.Bits`, desloca um número para a direita. A função `(.&.)`, do mesmo módulo, executa a conjunção (E) bit a bit de dois valores:

```
ghci> import Data.Bits
ghci> 0x10000 `shiftR` 4
4096
```

Agora que escrevemos `smallHex` e `astral`, podemos fornecer a definição de `hexEscape` (a função `ord`, do módulo `Data.Char`, converte um caractere para seu código numérico):

```haskell
-- src/PrettyJSON.hs
hexEscape :: Char -> Doc
hexEscape c | d < 0x10000 = smallHex d
            | otherwise   = astral (d - 0x10000)
  where d = ord c
```

Ok, agora pode compilar:

```
$ stack build
```

## Arrays, objetos e o cabeçalho do módulo

Comparada com strings, a impressão agradável de arrays e objetos é fácil. Sabemos que ambos são visualmente similares: cada um inicia com um caractere de abertura, seguido por uma série de valores separados por vírgulas, seguida por um caractere de fechamento. Vamos escrever uma função que captura essa estrutura comum:

```haskell
-- src/PrettyJSON.hs
series :: Char -> Char -> (a -> Doc) -> [a] -> Doc
series open close item = enclose open close
                       . fsep . punctuate (char ',') . map item
```

Comecemos interpretando o tipo dessa função. Ela recebe um caractere de abertura e um de fechamento, e uma função que sabe imprimir um valor de algum tipo desconhecido `a`, seguida por uma lista de valores do tipo `a`, e retorna um valor do tipo `Doc`.

Note que, embora a assinatura de tipos mencione quatro parâmetros, listamos apenas três na definição da função. Estamos simplesmente seguindo a mesma regra que nos permite simplificar uma definição como `myLength xs = length xs` para `myLength = length`.

Já escrevemos `enclose`. A função `fsep` viverá no módulo `Prettify`: ela combina uma lista de valores `Doc` em um só, possivelmente quebrando linhas caso a saída não caiba em uma linha só.

```haskell
-- src/Prettify.hs
fsep :: [Doc] -> Doc
fsep xs = undefined
```

A partir de agora, você já sabe definir os próprios esboços no `Prettify`, seguindo os exemplos anteriores — não mostraremos mais nenhum explicitamente.

A função `punctuate` também viverá no `Prettify`, e podemos defini-la **de verdade** em termos de funções para as quais já temos esboços:

```haskell
-- src/Prettify.hs
punctuate :: Doc -> [Doc] -> [Doc]
punctuate p []     = []
punctuate p [d]    = [d]
punctuate p (d:ds) = (d <> p) : punctuate p ds
```

Com essa definição de `series`, imprimir arrays é totalmente direto. Adicionamos esta equação ao final do bloco que escrevemos para `renderJValue`:

```haskell
-- src/PrettyJSON.hs
renderJValue (JArray ary) = series '[' ']' renderJValue ary
```

Para imprimir um objeto, precisamos de só um pouco mais de trabalho: para cada elemento, temos um nome **e** um valor com que lidar.

```haskell
-- src/PrettyJSON.hs
renderJValue (JObject obj) = series '{' '}' field obj
    where field (name,val) = string name
                          <> text ": "
                          <> renderJValue val
```

Ok, agora pode compilar:

```
$ stack build
```

### Escrevendo o cabeçalho do módulo

Agora que escrevemos o corpo do `PrettyJSON.hs`, voltamos ao topo e completamos a declaração do módulo:

```haskell
-- src/PrettyJSON.hs
module PrettyJSON
    (
      renderJValue
    ) where

import Prelude hiding ((<>))

import Numeric (showHex)
import Data.Char (ord)
import Data.Bits (shiftR, (.&.))

import SimpleJSON (JValue(..))
import Prettify (Doc, (<>), char, double, fsep, hcat, punctuate, text)
```

Exportamos apenas uma função deste módulo: `renderJValue`, nossa função de renderização de JSON. As outras definições existem puramente para dar suporte a ela, então não há razão para torná-las visíveis a outros módulos.

Sobre as importações: os módulos `Numeric`, `Data.Char` e `Data.Bits` são distribuídos junto com o GHC (no pacote `base`). Nós mesmos escrevemos o `SimpleJSON` e preenchemos o `Prettify` com definições esqueléticas. Note que não há diferença alguma na forma de importar módulos padrão e módulos que escrevemos.

!!! tip
    O livro original também importava `compact` e `pretty` neste cabeçalho. Não faça isso: essas funções serão **usadas por quem chama** o `PrettyJSON` (nós, no GHCi), não por ele — e o GHC moderno, com os avisos que o template do Stack ativa, reclamaria (com razão) de importação não utilizada.

Em cada diretiva `import` listamos explicitamente os nomes que queremos trazer para o escopo. Isso não é obrigatório — omitindo a lista, todos os nomes exportados ficam disponíveis —, mas é geralmente uma boa ideia:

- Uma lista explícita deixa claro **de onde** cada nome vem, facilitando achar a documentação de uma função desconhecida.
- Se o mantenedor de uma biblioteca remover ou renomear uma função, o erro de compilação resultante pode ocorrer muito tempo depois de escrevermos o módulo. A lista explícita age como lembrete de onde o nome ausente vinha, acelerando o diagnóstico.
- Se alguém adicionar a um módulo um nome idêntico a um do nosso código, sem lista explícita terminaremos com o mesmo nome em escopo duas vezes — e o GHC reportará ambiguidade se o usarmos. (Foi exatamente o que aconteceu, em escala global, com o `<>` e o Prelude!)

A explicitação das importações é uma orientação de bom senso, não uma regra rígida. Às vezes precisamos de tantos nomes de um módulo que listá-los se torna cansativo; em outros casos, um módulo é tão amplamente usado que qualquer programador Haskell experiente sabe o que vem dele.

