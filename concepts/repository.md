# Repository

> Conceito do Meta Platform Open Standard. Veja também
> [Package](./package.md), [Module / Layer / Group](./module-layer-group.md) e a
> especificação [Repository Metadata Standard](../specifications/repository-metadata-standard.md).

## Definição

Um **Repository** (repositório) é a **unidade distribuível e versionável** que
agrupa [packages](./package.md) do Meta Platform. Tipicamente é um repositório
Git, instalável em um [Ecosystem](#) a partir de uma **fonte** (`source`).

Um Repository **não é um projeto isolado**: ele é uma peça de um ecossistema. Os
seus packages se referenciam entre si e referenciam packages de **outros
repositórios** por **namespace** (nunca por caminho de arquivo), e são
executados pelo runtime comum da plataforma.

## Estrutura

```
<repository>/
├── README.md
├── metadata/
│   └── applications.json     # executáveis/aplicações publicados pelo repositório
└── <Nome>.Module/            # 1..N módulos
    └── <Nome>.layer/         # 1..N layers
        ├── <pkg>.<tipo>/     # packages (folhas)
        └── <Nome>.group/     # (opcional) agrupa packages de uma aplicação
            └── <pkg>.<tipo>/
```

A hierarquia interna (`Module → Layer → Group → Package`) está detalhada em
[Module / Layer / Group](./module-layer-group.md). Os sufixos de pasta
(`.Module`, `.layer`, `.group`) e os tipos de package aceitos são definidos em
[`ecosystem-defaults.json`](../specifications/metadados/ecosystem-defaults.json).

## Metadados

O metadado canônico de um repositório é
[`metadata/applications.json`](../specifications/repository-metadata-standard.md),
que lista os **executáveis/aplicações** que o repositório publica para o
ecossistema (associando `executable`, `packageNamespace` e
`supervisorSocketFileName`).

## Fontes (sources) e instalação

Um repositório é descoberto e instalado a partir de uma **fonte** registrada no
`sources.json` do ecossistema. Os tipos de fonte (`sourceType`) suportados são
`LOCAL_FS`, `GITHUB_RELEASE` e `GOOGLE_DRIVE`. A instalação é feita pela CLI
`repo` (`repository-manager.cli`) ou pelo
[Setup Wizard](https://github.com/Meta-Platform/meta-platform-setup-wizard-command-line).
Uma vez instalado, o repositório fica em `EcosystemData/repos/` (ver
[Ecosystem Data Directory Hierarchy Standard](../specifications/ecosystem-data-directory-hierarchy-standard.md)).

## Repositórios oficiais

| Repositório | Papel no ecossistema |
|-------------|----------------------|
| [meta-platform-essential-repository](https://github.com/Meta-Platform/meta-platform-essential-repository) | Runtime e bibliotecas essenciais (task executor, task loaders, libs comuns) + CLIs `repo`/`supervisor`. |
| [meta-platform-ecosystem-core-repository](https://github.com/Meta-Platform/meta-platform-ecosystem-core-repository) | Núcleo operacional: gerenciador de instâncias, serviços e painéis. |
| [meta-platform-applications-repository](https://github.com/Meta-Platform/meta-platform-applications-repository) | Aplicações de usuário final construídas sobre a plataforma. |
