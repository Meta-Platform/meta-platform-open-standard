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

Cada unidade do `execution-params.json` tem um `objectLoaderType`, resolvido por um
package **`.taskLoader`** correspondente. O mapa é montado **dinamicamente** pelo
`taskloader-registry.lib` (no essential) a partir do `taskloaders.json` de cada
repositório instalado — por isso os loaders são **distribuídos** por repositório:

| `objectLoaderType` | Task loader (package) | Repositório | Papel |
|--------------------|-----------------------|-------------|-------|
| `install-nodejs-package-dependencies` | `install-nodejs-package-dependencies.taskLoader` | EssentialRepo | Instala dependências Node.js do package. |
| `nodejs-package` | `nodejs-package.taskLoader` | EssentialRepo | Carrega o package Node.js. |
| `application-instance` | `application-instance.taskLoader` | EssentialRepo | Instancia uma aplicação (com serviços filhos). |
| `service-instance` | `service-instance.taskLoader` | EssentialRepo | Instancia um serviço. |
| `command-application` | `command-application.taskLoader` | EssentialRepo | Instancia uma aplicação de linha de comando. |
| `endpoint-instance` | `endpoint-instance.taskLoader` | EcosystemCoreRepo | Monta um endpoint HTTP / interface web. |
| `desktop-window-instance` | `desktop-window-instance.taskLoader` | PlatformApplicationsRepo | Abre uma janela Electron para packages `.desktopapp` (`loadURL`/`loadFile`/gui-host). |

Detalhes e exemplos de parâmetros em
[Tipos de Object Loader](../concepts/tipos-de-object-loader.md). Para o passo a
passo de **como criar um novo object loader / task loader**, veja o
[Guia: como criar e usar um Object Loader](https://github.com/Meta-Platform/meta-platform-essential-repository/blob/main/Runtime.Module/Executor.layer/task-executor.lib/docs/guia-criar-object-loader.md)
(implementação de referência).

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
