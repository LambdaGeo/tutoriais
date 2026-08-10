# Cobertura de Testes com HPC e Exercícios Finais

## Medindo a cobertura de testes com HPC

_(Esta seção foi inteiramente reescrita para as ferramentas atuais, e o relatório abaixo foi gerado de verdade sobre a nossa biblioteca.)_

Nossa suíte passa em todos os testes. Mas... ela testa **o quê**, exatamente? Essa pergunta tem uma resposta objetiva.

O **HPC** (_Haskell Program Coverage_) é um recurso do GHC que instrumenta o código para observar quais partes dele foram **realmente executadas** durante uma execução do programa. No contexto de testes, isso nos permite ver com precisão quais funções, ramos e expressões foram avaliados pela suíte — e, mais importante, quais **não** foram. O resultado é um conhecimento exato do percentual de código coberto, e o HPC ainda gera páginas HTML com o código-fonte colorido, facilitando localizar os pontos fracos da suíte.

Com o Stack, obter os dados de cobertura é um parâmetro a mais:

```
$ stack test --coverage
```

A suíte executa normalmente (todas as propriedades passando, como antes) e, ao final, o Stack imprime o relatório e os caminhos dos arquivos HTML gerados:

```
Generating coverage report for hs2json's test-suite "hs2json-test"

 19% expressions used (30/154)
  0% boolean coverage (0/3)
       0% guards (0/3), 3 unevaluated
     100% 'if' conditions (0/0)
     100% qualifiers (0/0)
 23% alternatives used (8/34)
  0% local declarations used (0/4)
 45% top-level declarations used (10/22)

The coverage report for hs2json's test-suite "hs2json-test" is available at
.../.stack-work/install/.../hpc/hs2json/hs2json-test/hpc_index.html
```

_(Os números referem-se ao módulo `Prettify`; por padrão, o Stack reporta a cobertura do código do **pacote** exercido pelos testes. Abra o `hpc_index.html` indicado no navegador para a versão visual.)_

!!! tip
    **Sem o Stack:** o HPC é do próprio GHC, então o fluxo manual equivalente é compilar com o flag `-fhpc`, executar o programa (o que gera um arquivo `.tix` com as contagens) e então usar o utilitário `hpc`: `hpc report` para o resumo textual e `hpc markup` para as páginas HTML. O `stack test --coverage` faz exatamente isso por você.

### Lendo o relatório

Aprender a ler essas linhas é o que dá valor à ferramenta:

| Métrica                       | O que mede                                                                                                 | Nosso resultado     |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------ | -------------------- |
| `expressions used`            | Quantas expressões do código foram avaliadas ao menos uma vez. É a métrica mais fina.                      | **19%** (30 de 154) |
| `boolean coverage` / `guards` | Dos pontos de decisão booleanos (guardas, `if`), quantos foram avaliados — e para os dois lados.           | **0%** (0 de 3)     |
| `alternatives used`           | Das alternativas de casamento de padrões (equações de função, ramos de `case`), quantas foram exercitadas. | **23%** (8 de 34)   |
| `local declarations`          | Definições em `where`/`let` executadas.                                                                    | **0%** (0 de 4)     |
| `top-level declarations`      | Funções de topo do módulo executadas.                                                                      | **45%** (10 de 22)  |

À primeira vista, os números parecem contraditórios: como uma suíte que "passa em tudo" cobre só 19% das expressões? A resposta está na visão por declaração. Abrindo o `hpc_index_fun.html` (ou o fonte anotado `Prettify.hs.html`, onde o código nunca executado aparece **destacado em amarelo**), o padrão salta aos olhos — as funções jamais tocadas pela suíte são:

```
fsep, (</>), softline, group, flatten, fits, compact, pretty
```

Ou seja: testamos completamente a metade **de construção** da biblioteca (`empty`, `char`, `text`, `double`, `line`, `<>`, `hcat`, `fold`, `punctuate`), mas **zero** da metade de **renderização** — justamente as funções mais complexas, `compact` e `pretty`, com seus `where` internos (eis os `local declarations 0%`: `transform`, `best`, `nicest`...) e suas guardas (eis o `boolean coverage 0%`: as guardas de `fits` e `nicest`). A suíte verde estava nos contando só metade da história — e o HPC expôs isso em uma linha. _(Curiosidade: as 22 declarações de topo contadas incluem os métodos gerados pelo `deriving` — `show`, `showsPrec`... —, que o HPC também rastreia.)_

### Fechando o ciclo: da lacuna à propriedade

O relatório não é um fim; é o começo da próxima iteração. Vamos escrever uma propriedade que exercite a renderização — um teste baseado em modelo minúsculo para a `compact`: renderizar compactamente um documento de texto puro deve devolver a própria string:

```haskell
-- test/Spec.hs
prop_compact_text s = compact (text s) == s
```

Rodando de novo com cobertura:

```
$ stack test --coverage

=== prop_compact_text from test/Spec.hs:45 ===
+++ OK, passed 100 tests.

Passou em todos os testes.

 27% expressions used (42/154)
 ...
 35% alternatives used (12/34)
 25% local declarations used (1/4)
 50% top-level declarations used (11/22)
```

Uma propriedade de uma linha: expressões cobertas de 19% para **27%**, alternativas de 23% para **35%**, e a `transform` interna da `compact` saiu do zero. É esse o ritmo do desenvolvimento guiado por propriedades **e** cobertura: a suíte diz _"o que testei está correto"_; o HPC diz _"eis o que você ainda não testou"_; e cada lacuna vira a próxima invariante.

!!! warning
    Cobertura alta **não** prova corretude — mede apenas o que foi _executado_, não o que foi _verificado_. Uma propriedade frouxa pode executar tudo e não conferir nada. Use os dois sinais juntos: propriedades fortes para a corretude, cobertura para achar os pontos cegos.

## Exercícios

**1.** Escreva propriedades para a metade ainda descoberta da biblioteca e acompanhe a cobertura subindo. Sugestões, em dificuldade crescente:

- `pretty` de um documento sem `softline` não depende da largura: `pretty w (text s)` deve ser igual a `s` para qualquer `w`;
- toda linha de `pretty w d` "cabe" — relacione com a intuição da função `fits` (cuidado: quando um pedaço **não cabe** de jeito nenhum, a linha pode estourar `w`; a propriedade precisa levar isso em conta);
- um teste baseado em modelo relacionando `compact` e `pretty`: os dois devem produzir o **mesmo texto**, a menos de espaços em branco e quebras de linha (comece definindo essa noção de equivalência!).

**2.** Implemente o método `shrink` na instância `Arbitrary Doc` (investigue `genericShrink`, que exige `deriving (Generic)` no tipo) e provoque uma falha de propósito para ver o QuickCheck reduzir o contraexemplo ao mínimo.

**3.** Nosso gerador de `Doc` escolhe entre os seis construtores com igual probabilidade, e os casos recursivos podem, ocasionalmente, gerar árvores enormes. Investigue as funções `sized` e `frequency` do QuickCheck e reescreva o gerador limitando a profundidade da árvore pelo "tamanho" do teste.

---

_Baseado nos Capítulos 5 e 11 de **Real World Haskell**, copyright 2007, 2008 Bryan O'Sullivan, Don Stewart e John Goerzen, sob licença Creative Commons Attribution-Noncommercial 3.0. Tradução do projeto rwh-ptbr; revisão, atualização para GHC 9.x/Stack/QuickCheck 2.14 e validação de todo o código nesta edição v2._
