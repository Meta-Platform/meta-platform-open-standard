# Environment Runtime Standard

Define como um [Runtime Environment](../concepts/runtime-environment.md) é
executado: a tradução do grafo de metadados em plano de execução e a execução
desse plano pelo **Task Executor** via **object loaders**.

## Entradas e saídas

| Artefato | Papel |
|----------|-------|
| `metadata-hierarchy.json` | Grafo de metadados (package raiz + dependências resolvidas por namespace). |
| `execution-params.json` | Plano de execução derivado do grafo. Ver [Execution Params Standard](./packages/execution-params-standard.md). |
| instância do package | Resultado: serviços/endpoints/CLI ativos. |

## Pipeline (implementação de referência)

A sequência executada pelo
[package-executor](https://github.com/Meta-Platform/meta-platform-package-executor-command-line)
(`ExecutePackage`):

1. **Listagem de packages** dos repositórios instalados (`ListPackages`).
2. **Construção do grafo** (`BuildMetadataHierarchy`) a partir do `packagePath`,
   resolvendo dependências por namespace → `metadata-hierarchy.json`.
3. **Criação do ambiente** (`CreateEnvironment` + `PrepareDataDir`): diretório
   `nome+hash` em `environments/` e `.dependencies/`.
4. **Tradução** (`TranslateMetadataHierarchyForExecutionParams`) →
   `execution-params.json`.
5. **Task Executor** (`task-executor.lib`): cria as tasks a partir do plano e as
   executa respeitando dependências e regras de ativação.

## Object loaders

Cada unidade do `execution-params.json` tem um `objectLoaderType`, resolvido por
um **task loader** correspondente (no
[essential-repository](https://github.com/Meta-Platform/meta-platform-essential-repository),
layer `EssentialTaskLoaders.layer`):

| `objectLoaderType` | Task loader | Papel |
|--------------------|-------------|-------|
| `install-nodejs-package-dependencies` | `install-nodejs-package-dependencies.lib` | Instala dependências Node.js do package. |
| `nodejs-package` | `nodejs-package.lib` | Carrega o package Node.js. |
| `application-instance` | `application-instance.lib` | Instancia uma aplicação (com serviços filhos). |
| `service-instance` | `service-instance.lib` | Instancia um serviço. |
| `endpoint-instance` | `endpoint-instance.lib` | Monta um endpoint HTTP / interface web. |
| `command-application` | `command-application.lib` | Instancia uma aplicação de linha de comando. |

Detalhes e exemplos de parâmetros em
[Tipos de Object Loader](../concepts/tipos-de-object-loader.md).

## Ciclo de vida das tasks

Cada task percorre os estados de `TaskStatus`
(`AWAITING_PRECONDITIONS → … → ACTIVE/FINISHED`, ou `FAILURE`), observáveis pela
interface de supervisão (ver
[Supervisor Socket Standard](./supervisor-socket-standard.md)). O package é
considerado "no ar" quando suas tasks estão `ACTIVE`/`FINISHED` sem `FAILURE`.

## Ativação e ligação entre unidades

- `activationRules` — condições para uma unidade ser ativada.
- `linkedParameters` / `agentLinkRules` — ligação a outras unidades em runtime
  (ver [Dependency Resolution Standard](./dependency-resolution-standard.md)).
