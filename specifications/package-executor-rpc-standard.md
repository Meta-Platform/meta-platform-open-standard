# Package Executor RPC Standard

Define a interface **gRPC de supervisão** exposta por todo processo iniciado pelo
[Package Executor](../concepts/executable.md). A definição canônica é o arquivo
[`proto/package_executor_rpc.proto`](../proto/package_executor_rpc.proto); este
documento descreve seu contrato. O **transporte** (socket Unix, localização,
ciclo de vida) está em [Supervisor Socket Standard](./supervisor-socket-standard.md).

- **Package proto:** `PackageExecutorRPCSpec`
- **Serviço:** `PackageExecutorRPCService`

## Serviço `PackageExecutorRPCService`

| RPC | Requisição | Resposta | Descrição |
|-----|------------|----------|-----------|
| `KillInstance` | `Empty` | `Empty` | Encerra o processo supervisionado. |
| `GetStartupArguments` | `Empty` | `StartupArgumentsResponse` | Argumentos com que o package foi iniciado. |
| `GetProcessInformation` | `Empty` | `ProcessInformationResponse` | PID, plataforma e arquitetura. |
| `GetStatus` | `Empty` | `ExecutionStatusResponse` | Status atual da execução. |
| `ListTasks` | `Empty` | `TaskListResponse` | Lista as tasks (unidades de execução). |
| `GetTask` | `TaskRequest` | `TaskInformation` | Detalhes de uma task por `taskId`. |
| `LogStreaming` | `Empty` | `stream LogResponse` | Streaming dos logs do processo. |
| `StatusChangeNotification` | `Empty` | `stream ExecutionStatusResponse` | Notifica mudanças de status. |

## Enums

### `ExecutionStatus`

| Valor | # | Significado |
|-------|---|-------------|
| `UNKNOWN` | 0 | Desconhecido. |
| `WAITING_FOR_FIRST_CONNECTION` | 1 | Aguardando a primeira conexão. |
| `STARTING` | 2 | Iniciando. |
| `RUNNING` | 3 | Em execução. |
| `ERROR` | 4 | Em erro. |

### `TaskStatus`

| Valor | # | Significado |
|-------|---|-------------|
| `AWAITING_PRECONDITIONS` | 1 | Aguardando pré-condições. |
| `PRECONDITIONS_COMPLETED` | 2 | Pré-condições satisfeitas. |
| `PREPPED_TO_START` | 3 | Pronta para iniciar. |
| `STARTING` | 4 | Iniciando. |
| `ACTIVE` | 5 | Ativa. |
| `STOPPING` | 6 | Parando. |
| `FINISHED` | 7 | Concluída. |
| `FAILURE` | 8 | Falhou. |
| `TERMINATED` | 9 | Encerrada. |

## Mensagens

### `StartupArgumentsResponse`
`packagePath`, `startupJsonFilePath`, `ecosystemDefaultJsonFilePath`,
`nodejsProjectDependenciesPath`, `verbose` (bool), `supervisorSocketPath`,
`executableName`, `commandLineArgs`, `ecosystemDataPath`.

### `ProcessInformationResponse`
`pid` (int32), `platform` (string), `arch` (string).

### `ExecutionStatusResponse`
`status` (`ExecutionStatus`).

### `TaskListResponse`
`tasksList` (repetido `Task`).

### `Task`
`taskId` (int32), `pTaskId` (`Int32Value`, id da task pai), `objectLoaderType`
(string), `status` (`TaskStatus`), `staticParameters` (`Struct`).

### `TaskInformation`
Tudo de `Task` mais: `hasChildTasks` (bool), `linkedParameters` (`Struct`),
`agentLinkRules` (repetido `AgentLinkRules`), `activationRules` (`Struct`). Esses
campos espelham o [Execution Params Standard](./packages/execution-params-standard.md).

### `AgentLinkRules`
`referenceName` (string), `requirement` (`Struct`).

### `TaskRequest`
`taskId` (int32).

### `LogResponse`
`sourceName` (string), `type` (string), `message` (string).

> O campo `EventResponse { event_message }` existe no `.proto` mas não é
> referenciado por nenhum RPC atualmente. `> TODO: confirmar` se é reservado para
> uso futuro ou pode ser removido.

## Implementação de referência

Cliente/servidor no
[package-executor](https://github.com/Meta-Platform/meta-platform-package-executor-command-line)
(`CreateBinaryInterfaceViaSocket`) e no `instance-supervisor.cli` /
`supervisor.lib` do
[essential-repository](https://github.com/Meta-Platform/meta-platform-essential-repository).
