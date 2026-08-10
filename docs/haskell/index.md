# Construindo e Testando uma Biblioteca Haskell: JSON, Pretty Printing e QuickCheck

*Um guia prático e completo adaptado de Real World Haskell sobre módulos, tipos algébricos, design de bibliotecas e testes com QuickCheck e HPC.*

---

Este guia reúne, em duas partes, os capítulos 5 e 11 de _Real World Haskell_ (Bryan O'Sullivan, Don Stewart e John Goerzen, 2008), em adaptação para o português. Na **Parte 1**, construímos do zero uma biblioteca de manipulação e impressão agradável (_pretty printing_) de dados JSON — aprendendo, no caminho, módulos, tipos algébricos, tipos abstratos e o design incremental de bibliotecas. Na **Parte 2**, colocamos essa biblioteca à prova: especificamos suas invariantes com o **QuickCheck**, deixamos a máquina gerar milhares de casos de teste, e medimos a cobertura com o **HPC** — que revela exatamente o que a suíte deixou de fora.

!!! note
    **Nota desta edição revisada (v2).** O texto original tem mais de quinze anos, e o ecossistema Haskell mudou bastante. Nesta revisão: (1) instalação e criação de projetos atualizadas para o fluxo moderno com **GHCup** e **Stack** (`package.yaml`/hpack); (2) código ajustado para **GHC 9.x** — em particular, desde o GHC 8.4 o operador `<>` faz parte do Prelude, o que exige um ajuste que o livro não previa; (3) as saídas do QuickCheck 2.14 atualizadas, a seção do `quickCheckAll` reescrita (com uma correção no código de saída da suíte) e a seção do **HPC** refeita para as ferramentas atuais; e (4) **todo o código e todas as saídas de terminal foram executados e conferidos** com GHC 9.4 e QuickCheck 2.14.3.

---

### 📚 Índice

**Parte 1 — Escrevendo uma biblioteca para dados no formato JSON**

- [Tour pelo JSON e Preparação do Ambiente](01-tour-json-ambiente.md)
- [Representando JSON em Haskell: Módulos e Compilação](02-representando-json.md)
- [Imprimindo e Renderizando Dados JSON](03-renderizacao-simples.md)
- [Impressão Agradável de Strings, Arrays e Objetos](04-impressao-strings-arrays.md)
- [Construindo a Biblioteca Prettify](05-biblioteca-prettify.md)
- [Exercícios, Empacotamento e Leitura Adicional](06-pacote-exercicios.md)

**Parte 2 — Testes e garantia de qualidade**

- [Preparando o Projeto e QuickCheck Básico](07-quickcheck-intro.md)
- [Estudo de Caso: Especificando um Pretty Printer](08-estudo-caso-pretty-printer.md)
- [Cobertura de Testes com HPC e Exercícios Finais](09-cobertura-hpc.md)
