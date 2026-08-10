# Introdução ao Docker

![image.png](img/index-image.png)

### Chega de "Na minha máquina funciona": Uma Introdução ao Docker

Você já perdeu horas configurando um ambiente de desenvolvimento do zero? Ou pior, terminou um projeto que rodava perfeitamente no seu computador, mas quando enviou para o colega (ou para produção), ouviu a temida frase: *"Não está rodando aqui"*?

Se você já passou por essas dores de cabeça (como problemas de versão, dependências que conflitam ou ambientes difíceis de reproduzir), você vai entender rapidamente o valor do Docker.

O objetivo aqui é simples: garantir que o seu código rode **exatamente da mesma forma em qualquer lugar**.

### Máquina Virtual (VM) vs Container: Qual a diferença?

Para resolver o problema do isolamento de ambiente, a solução clássica sempre foi criar uma Máquina Virtual (VM). Mas os containers trouxeram uma abordagem muito mais eficiente.

Para entender a diferença, vamos usar uma analogia simples:

- **A Máquina Virtual (VM) é como construir uma casa independente.** Ela precisa do seu próprio terreno, gerador de energia, encanamento e fundação. No mundo da computação, isso significa emular um hardware inteiro e instalar um Sistema Operacional (SO) completo para cada aplicação. O resultado? É pesado, consome muitos recursos da sua máquina e demora minutos para iniciar (boot lento).
- **O Container é como alugar um apartamento em um prédio.** Ele tem seu espaço totalmente isolado e privado, mas compartilha a infraestrutura básica do condomínio (água, luz, portaria). Na computação, o container usa o núcleo (kernel) do sistema operacional que já está rodando no seu computador. Ele isola apenas o processo da sua aplicação. O resultado? É extremamente leve e inicia em questão de milissegundos.

### Imagem ≠ Container

Um dos pontos que mais confunde quem está começando é a diferença entre *Imagem* e *Container*. A regra de ouro é:

- **A Imagem é a receita.** Ela é um arquivo contendo um conjunto de instruções e dependências (camadas apenas de leitura/read-only). Ela diz exatamente o que precisa existir no ambiente.
- **O Container é o bolo pronto.** É a execução da receita na vida real. Quando você roda uma imagem, o Docker adiciona uma camada gravável (onde a aplicação pode salvar arquivos temporários, etc) e inicia o processo.

A partir de uma única "receita" (uma imagem), você pode criar infinitos "bolos" (containers) idênticos e independentes, rodando ao mesmo tempo.

---

[Docker CLI Puro: Exploração Guiada](docker-cli-exploracao-guiada.md)

[Ciclo de Vida, Efemeridade & Volumes ](ciclo-de-vida-efemeridade-volumes.md)

[Tradução CLI → Docker Compose (Construção Passo a Passo)](traducao-cli-docker-compose.md)

[Otimização de Imagens + Estratégia Dev vs Prod](otimizacao-imagens-dev-vs-prod.md)

### Mais

[🏠 TAREFA PARA CASA: Polyglot Full-Stack com Docker Compose](tarefa-polyglot-full-stack.md)

[🐳 DOCKER & DOCKER COMPOSE – CHEAT SHEET](cheat-sheet-docker-compose.md)
