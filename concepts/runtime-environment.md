# Runtime Environment

> Conceito do Meta Platform Open Standard. Veja também
> [Package](./package.md), [Executable](./executable.md) e as especificações
> [Environment Runtime Standard](../specifications/environment-runtime-standard.md)
> e [Ecosystem Data Directory Hierarchy Standard](../specifications/ecosystem-data-directory-hierarchy-standard.md).

## Definição

Um **Runtime Environment** (ambiente de execução) é o **diretório isolado** que a
plataforma cria para **cada execução de um package**. Ele contém tudo o que o
package precisa para rodar: o grafo de metadados, o plano de execução e as
dependências resolvidas.

Os ambientes ficam em `EcosystemData/environments/` e são nomeados com o **nome
do package + um hash do seu caminho** no sistema de arquivos — por exemplo:

```
repository-manager.cli-e9766fee7f2d40e9c498cabb6c06a9581236c7ba9d1310999b5e24d1dd126899
```

## O que tem dentro

| Item | Conteúdo |
|------|----------|
| `metadata-hierarchy.json` | O grafo completo de metadados (o package raiz + todas as dependências resolvidas por namespace) que precisa ser carregado. |
| `execution-params.json` | O **plano de execução**: as unidades a executar (object loaders) e suas interdependências, consumido pelo [Task Executor](../specifications/environment-runtime-standard.md). |
| `.dependencies/node_modules` | As dependências Node.js instaladas para o ambiente. |

Os nomes desses arquivos/diretórios vêm de
[`ecosystem-defaults.json`](../specifications/metadados/ecosystem-defaults.json)
(`ECOSYSTEMDATA_CONF_FILENAME_PKG_GRAPH_DATA`,
`ECOSYSTEMDATA_CONF_FILENAME_EXECUTION_PLAN_DATA`,
`EXECUTIONDATA_CONF_DIRNAME_DEPENDENCIES`).

## Como é criado (fluxo)

1. O **Package Executor** monta o `metadata-hierarchy.json` resolvendo as
   dependências do package por namespace (ver
   [Dependency Resolution Standard](../specifications/dependency-resolution-standard.md)).
2. Cria o diretório do ambiente (`nome + hash`) e prepara `.dependencies`.
3. Traduz o grafo de metadados em `execution-params.json` (ver
   [Execution Params Standard](../specifications/packages/execution-params-standard.md)).
4. O **Task Executor** lê o `execution-params.json` e instancia cada unidade via
   o **object loader** correspondente (ver
   [Tipos de Object Loader](./tipos-de-object-loader.md)), produzindo a
   **instância do package**.
5. Se houver supervisão, o processo expõe um
   [supervisor socket](../specifications/supervisor-socket-standard.md).

## Por que isolar

- **Reprodutibilidade:** cada execução tem suas dependências e seu plano.
- **Coexistência:** o mesmo package pode rodar em contextos diferentes (o hash
  do caminho diferencia os ambientes).
- **Supervisão:** o ambiente é a fronteira observável pelo
  [Task Executor](../specifications/environment-runtime-standard.md) e pela
  interface de supervisão.
