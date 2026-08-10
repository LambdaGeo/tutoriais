# Contribuindo com novos tutoriais

Este repositório hospeda múltiplos tutoriais independentes (Docker é o primeiro). Para adicionar um novo tutorial, ele deve atender aos critérios abaixo.

## Critérios de entrada

- **Autocontido e independente de disciplina.** O tutorial não pode assumir que o leitor já cursou uma disciplina completa da coleção — só os pré-requisitos técnicos explicitados no próprio tutorial (ex: "conhecimento básico de terminal").
- **Progressão didática.** Seguir a estrutura já estabelecida pelo tutorial de Docker: exploração guiada → conceitos → prática guiada → tarefa final.
- **Cheat sheet recomendada, não obrigatória.** Uma página de referência rápida ao final ajuda, mas não é bloqueante para aceitar o tutorial.
- **Build limpo.** `mkdocs build --strict` deve passar sem erros nem warnings antes de qualquer PR ser aceito — mesmo gate usado nas outras disciplinas da coleção.
- **Reprodutibilidade.** Todo comando e passo deve ser executável exatamente como escrito, sem depender de estado implícito não declarado entre seções.

## Estrutura de pastas

Cada tutorial vive em sua própria pasta dentro de `docs/`, com uma `index.md` de entrada e uma subpasta `img/` própria para suas imagens:

```
docs/
├── index.md              # hub — não editar diretamente ao adicionar tutorial,
│                          # só adicionar uma entrada nova na lista
└── <nome-do-tutorial>/
    ├── index.md
    ├── img/
    └── ... demais capítulos
```

## Passos para adicionar um tutorial

1. Criar a pasta `docs/<nome-do-tutorial>/` com o conteúdo e imagens.
2. Adicionar uma seção nova no `nav` do `mkdocs.yml`, seguindo o padrão já usado para Docker.
3. Adicionar uma entrada na lista em `docs/index.md` (título, uma ou duas frases de escopo, link para a `index.md` do tutorial).
4. Rodar `mkdocs build --strict` localmente antes de abrir o PR.
