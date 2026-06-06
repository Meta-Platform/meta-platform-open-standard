# Supervisor Socket Standard

Define o **transporte de supervisão** dos processos do Meta Platform: um
servidor **gRPC sobre socket Unix** que cada processo iniciado pelo
[Package Executor](../concepts/executable.md) pode expor. O **contrato** das
operações está em [Package Executor RPC Standard](./package-executor-rpc-standard.md).

## Onde fica o socket

Os sockets de supervisão ficam em `EcosystemData/supervisor-sockets/`
(`ECOSYSTEMDATA_CONF_DIRNAME_SUPERVISOR_UNIX_SOCKET_DIR` em
[`ecosystem-defaults.json`](./metadados/ecosystem-defaults.json)). O nome do
arquivo de socket de cada executável é definido em `applications.json`
(`supervisorSocketFileName`, ex.: `instance-manager.sock`) — ver
[Repository Metadata Standard](./repository-metadata-standard.md).

> Há também `EcosystemData/sockets/` para os sockets Unix de **IPC entre
> serviços** (HTTP/aplicação) — distinto dos sockets de **supervisão**.

## Transporte

- **Protocolo:** gRPC (`@grpc/grpc-js`) sobre **socket Unix** (`unix:<caminho>`).
- **Credenciais:** inseguras/locais (`createInsecure`) — supervisão é local à
  máquina.
- **Serviço:** `PackageExecutorRPCService` (ver
  [Package Executor RPC Standard](./package-executor-rpc-standard.md)).

## Ciclo de vida

1. Ao ser iniciado com `--supervisorSocket <caminho>`, o package-executor cria o
   servidor gRPC e faz `bind` no socket Unix.
2. Se o caminho usar o prefixo `unix:`, o arquivo de socket anterior é removido
   antes do `bind` (limpeza de socket órfão).
3. Enquanto o processo vive, clientes (ex.: CLI `supervisor`) conectam para
   consultar status, listar/inspecionar tasks, fazer streaming de log e encerrar
   a instância.
4. Ao encerrar, o arquivo de socket é removido (shutdown handler).

## Cliente de referência

A CLI `supervisor` (`instance-supervisor.cli`) e a `supervisor.lib` do
[essential-repository](https://github.com/Meta-Platform/meta-platform-essential-repository)
implementam o cliente:

```bash
supervisor sockets                      # lista os sockets de supervisão
supervisor status instance-manager.sock # status do processo
supervisor tasks  instance-manager.sock # tasks em execução
supervisor log    instance-manager.sock # streaming de log
supervisor kill   instance-manager.sock # encerra o processo
supervisor show task 46 --socket instance-manager.sock
```

## Estados observáveis

O socket expõe os estados de `ExecutionStatus` (processo) e `TaskStatus` (cada
task), definidos no
[Package Executor RPC Standard](./package-executor-rpc-standard.md).
