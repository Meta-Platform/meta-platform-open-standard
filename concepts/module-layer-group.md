# Module / Layer / Group

> Conceito do Meta Platform Open Standard. Veja também
> [Repository](./repository.md) e [Package](./package.md).

Dentro de um [Repository](./repository.md), os [packages](./package.md) não ficam
soltos: são organizados em uma hierarquia de **itens de organização** que dá
contexto e namespace a cada unidade.

```
Repository
└── Module   (*.Module)
    └── Layer (*.layer)
        ├── Package        (*.lib, *.cli, ...)   ← folha
        └── Group (*.group)
            └── Package
```

Os sufixos de pasta são definidos em
[`ecosystem-defaults.json`](../specifications/metadados/ecosystem-defaults.json)
(`REPOS_CONF_EXT_MODULE_DIR = "Module"`, `REPOS_CONF_EXT_LAYER_DIR = "layer"`,
`REPOS_CONF_EXT_GROUP_DIR = "group"`).

## Module (`*.Module`)

Divisão **macro de responsabilidade** dentro de um repositório. Convenções
observadas nos repositórios oficiais:

- **`Commons.Module`** — utilitários e bibliotecas compartilhadas.
- **`Runtime.Module`** — peças de tempo de execução (task executor, task
  loaders, helpers de metadados).
- **`Main.Module`** — as aplicações/CLIs/serviços principais do repositório.
- **`Apps.Module` / `Base.Module`** — aplicações de usuário e suas bases
  reutilizáveis (no applications-repository).

## Layer (`*.layer`)

Camada que agrupa packages da **mesma preocupação ou do mesmo tipo**. Exemplos
reais: `Application.layer`, `Libraries.layer`, `PlatformLibraries.layer`,
`Utilities.layer`, `Services.layer`, `Webservices.layer`, `Executor.layer`,
`MetadataHelpers.layer`, `Loaders.layer` e `Registry.layer` (do
`Taskloaders.Module`), `Admin.layer`, `Tools.layer`.

## Group (`*.group`)

Agrupa os packages que, **juntos, formam uma aplicação completa**. É comum uma
aplicação web ser composta por vários packages no mesmo group — por exemplo
`APIDesigner.group`, com quatro: `api-designer.webapp`, `api-designer.webgui`,
`api-designer.webservice` e `api-designer.desktopapp`. O Group é **opcional**:
packages independentes ficam direto na Layer.

## Por que essa hierarquia importa

- Dá **namespace e contexto** a cada package sem acoplá-lo ao disco.
- Permite que o runtime **localize e carregue** packages pela posição
  (`Module/Layer/[Group/]Package`) declarada em `applications.json` e nos
  metadados.
- Mantém o repositório **navegável e governável** à medida que cresce.
