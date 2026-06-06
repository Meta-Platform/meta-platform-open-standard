# Meta Platform Open Standard

A **Meta Platform** promove a atomização, modularização e componentização de
sistemas, permitindo soluções escaláveis, flexíveis, interoperáveis e fáceis de
manter. O **Open Standard** é a especificação aberta e **independente de
implementação** desse modelo: hierarquia de repositórios, metadados de pacotes,
ambiente de execução, supervisão e instalação.

Este repositório é a **referência canônica**. Para documentação prática
(instalação, uso, tutoriais), veja o
[portal da organização](https://github.com/Meta-Platform) e a pasta
[`docs/`](https://github.com/Meta-Platform/.github/tree/main/docs). Para o mapa de
repositórios, veja
[repository-map](https://github.com/Meta-Platform/.github/blob/main/docs/repository-map.md).

---

## Conceitos

Definições centrais, independentes de implementação:

- [Package](./concepts/package.md) — a unidade atômica da plataforma.
- [Repository](./concepts/repository.md) — a unidade distribuível que agrupa packages.
- [Module / Layer / Group](./concepts/module-layer-group.md) — a hierarquia de organização.
- [Executable](./concepts/executable.md) — pontos de entrada publicados por um package.
- [Runtime Environment](./concepts/runtime-environment.md) — o ambiente isolado de cada execução.
- [Metadata](./concepts/metadata.md) — visão de conceito dos metadados de package.
- [Tipos de Object Loader](./concepts/tipos-de-object-loader.md) — tipos de unidade instanciados pelo task executor.

## Especificações

**Metadados e estrutura**
- [Package Metadata Standard](./specifications/package-metadata-standard.md)
- [Executable Manifest Standard](./specifications/executable-manifest-standard.md)
- [Repository Metadata Standard](./specifications/repository-metadata-standard.md)

**Execução**
- [Execution Params Standard](./specifications/packages/execution-params-standard.md)
- [Environment Runtime Standard](./specifications/environment-runtime-standard.md)
- [Dependency Resolution Standard](./specifications/dependency-resolution-standard.md)

**Supervisão**
- [Package Executor RPC Standard](./specifications/package-executor-rpc-standard.md)
- [Supervisor Socket Standard](./specifications/supervisor-socket-standard.md)

**Ecossistema e instalação**
- [Ecosystem Data Directory Hierarchy Standard](./specifications/ecosystem-data-directory-hierarchy-standard.md)
- [Ecosystem Installation Profile Standard](./specifications/ecosystem-installation-profile-standard.md)

### Metadados e proto de referência

- [`specifications/metadados/ecosystem-defaults.json`](./specifications/metadados/ecosystem-defaults.json)
  — parâmetros padrão de um ecossistema (nomes de diretórios, extensões, etc.).
- [`proto/package_executor_rpc.proto`](./proto/package_executor_rpc.proto) —
  definição gRPC do serviço de supervisão.

---

## Como tudo se conecta

```
Installation Profile → Setup Wizard → EcosystemData → Repositories
   → applications.json → Executables → Package Executor
   → Runtime Environment → Task Executor → Package Instance → Supervisor Socket
```

Fluxo detalhado em
[execution-flow](https://github.com/Meta-Platform/.github/blob/main/docs/execution-flow.md).

## Tópicos em aberto

- **Empacotamento do ecossistema em container** — desenho ainda não especificado
  (`> TODO: confirmar`; origem: `notas.txt`).
