
# Tipos de Object Loader

> Conceito do Meta Platform Open Standard. Veja também
> [Runtime Environment](./runtime-environment.md), o
> [Environment Runtime Standard](../specifications/environment-runtime-standard.md)
> e o [Execution Params Standard](../specifications/packages/execution-params-standard.md).

## O que é um Object Loader

Um **object loader** é a unidade que sabe **como instanciar** um tipo de objeto
do plano de execução. O [Task Executor](../specifications/environment-runtime-standard.md)
orquestra as tasks (cria, espera pré-condições, ativa na ordem certa e acompanha
o status), mas delega ao object loader o trabalho concreto de cada unidade —
instalar dependências, carregar um pacote, subir um serviço, montar um endpoint
ou rodar uma CLI.

Cada unidade do `execution-params.json` declara um campo **`objectLoaderType`**
(uma string) que identifica qual object loader deve carregá-la. Na implementação
de referência, cada `objectLoaderType` é resolvido por um package do tipo
**[`.taskLoader`](./package.md)** — que vive no `Taskloaders.Module` do repositório
(no [essential-repository](https://github.com/Meta-Platform/meta-platform-essential-repository),
em `Taskloaders.Module/Loaders.layer`) e é declarado no
[`taskloaders.json`](../specifications/repository-metadata-standard.md) do repositório
(com o `objectLoaderType`, o package que o implementa e as dependências npm que ele
exige). Por isso os termos **object loader** (o tipo) e **task loader** (o package/função
que o implementa) são usados de forma intercambiável.

> Historicamente esses loaders eram packages `.lib` na layer `EssentialTaskLoaders.layer`;
> passaram a ser packages do tipo `.taskLoader` reunidos no `Taskloaders.Module`. A
> chave `objectLoaderType` permanece a mesma — mudou apenas o tipo/local do package.

| `objectLoaderType` | Papel | Detalhado em |
|--------------------|-------|--------------|
| `install-nodejs-package-dependencies` | Instala as dependências Node.js do package. | [↓](#install-nodejs-package-dependencies) |
| `nodejs-package` | Carrega o package Node.js e expõe um handler. | [↓](#nodejs-package) |
| `application-instance` | Instancia uma aplicação completa (com serviços filhos). | [↓](#application-instance) |
| `service-instance` | Instancia um serviço dentro de uma aplicação. | [↓](#service-instance) |
| `endpoint-instance` | Monta um endpoint HTTP (controller ou interface web). | [↓](#endpoint-instance) |
| `command-application` | Instancia uma aplicação de linha de comando (CLI). | [↓](#command-application) |
| `desktop-window-instance` | Abre uma janela Electron carregando conteúdo HTML local. | [↓](#desktop-window-instance) |

## Contrato e ciclo de vida (visão de implementação)

Na implementação de referência, um object loader é uma função
`(loaderParams, executorChannel) => getServiceObject` que registra o que fazer ao
iniciar/parar a task e reporta seu status. Os campos comuns do `execution-params`
(`staticParameters`, `linkedParameters`, `agentLinkRules`, `activationRules`,
`children`) estão no
[Execution Params Standard](../specifications/packages/execution-params-standard.md).

Para o **passo a passo de como criar um novo object loader / task loader**, veja o
[Guia: como criar e usar um Object Loader](https://github.com/Meta-Platform/meta-platform-essential-repository/blob/main/Runtime.Module/Executor.layer/task-executor.lib/docs/guia-criar-object-loader.md).

---

# `application-instance`

Define uma instância de aplicação completa com startup parameters e serviços filhos.

**Parâmetros:**
- `startupParams` (object): Parâmetros de inicialização
  - `socket` (string): Caminho do socket Unix de comunicação
  - `serverName` (string): Nome do servidor/aplicação
  - `serviceStorageFilePath` (string): Caminho do banco de dados SQLite
  - `servicesInstanceDataDirPath` (string): Diretório de dados da aplicação
- `namespace` (string): Identificador único
- `rootPath` (string): Caminho raiz da aplicação

**Exemplo no execution params:**
```json
{
  "objectLoaderType": "application-instance",
  "staticParameters": {
    "startupParams": {
      "socket": "/home/kadisk/EcosystemData/sockets/service-orchestrator.app.sock",
      "serverName": "ServiceOrchestratorAppInstance",
      "serviceStorageFilePath": "~/virtual-desk-state/local-databases/service-orchestrator-service.sqlite",
      "servicesInstanceDataDirPath": "~/virtual-desk-state/local-app-data/services-instance-data"
    },
    "namespace": "@/service-orchestrator.app",
    "rootPath": "/home/kadisk/EcosystemData/repos/KADISKCorpRepo/VirtualDesk.Module/PlatformApplications.layer/service-orchestrator.app"
  },
  "children": [ ... ]
}
```

---

# `service-instance`

Define uma instância de serviço dentro de uma aplicação.

**Parâmetros:**
- `tag` (string): Identificador único do serviço (ex: `"@@/server-service"`)
- `name` (string - opcional): Nome do serviço
- `port` (string - opcional): Porta ou socket do serviço
- `path` (string): Caminho do arquivo do serviço, resolvido via `nodejsPackageHandler`
- `localPath` (string - alternativa a `path`): Caminho carregado por `require`
  direto (sem `nodejsPackageHandler`); usado quando o serviço não vem de um
  package resolvido
- `serviceParameterNames` (array): Lista de nomes de parâmetros esperados
- `serviceStorageFilePath` (string - opcional): Caminho do banco de dados
- `instanceDataDirPath` (string - opcional): Diretório de dados da instância
- `nodejsPackageHandler` (string): Referência ao pacote Node.js (usado com `path`)

**Exemplo no execution params:**
```json
{
  "objectLoaderType": "service-instance",
  "staticParameters": {
    "tag": "@@/server-service",
    "name": "ServiceOrchestratorAppInstance",
    "port": "/home/kadisk/EcosystemData/sockets/service-orchestrator.app.sock",
    "path": "Services/HTTPServer.service",
    "serviceParameterNames": [
      "name",
      "port",
      "authenticationService"
    ]
  },
  "linkedParameters": {
    "nodejsPackageHandler": "@/server-manager.service"
  },
  "agentLinkRules": [
    {
      "referenceName": "@/server-manager.service",
      "requirement": {
        "&&": [
          {
            "property": "params.tag",
            "=": "@/server-manager.service"
          },
          {
            "property": "status",
            "=": "ACTIVE"
          }
        ]
      }
    }
  ]
}
```

---

# `endpoint-instance`

Define um endpoint HTTP com controller e API template associados.

**Parâmetros:**
- `url` (string): Caminho da rota (ex: `"/server-manager"`)
- `type` (string): Tipo do endpoint (ex: `"controller"`)
- `apiTemplate` (string): Caminho do arquivo de template API
- `controller` (string): Caminho do arquivo do controller
- `nodejsPackageHandler` (string): Pacote que fornece o endpoint
- `controllerParams` (object): Parâmetros passados ao controller
- `serverService` (string): Referência ao serviço de servidor

**Exemplo no execution params:**
```json
{
  "objectLoaderType": "endpoint-instance",
  "staticParameters": {
    "url": "/server-manager",
    "type": "controller",
    "apiTemplate": "APIs/HTTPServers.api.json",
    "controller": "Controllers/HTTPServers.controller"
  },
  "linkedParameters": {
    "nodejsPackageHandler": "@/server-manager.webservice",
    "controllerParams": {
      "httpServerService": "@@/server-service"
    },
    "serverService": "@@/server-service"
  },
  "agentLinkRules": [ ... ]
}
```

---

# `install-nodejs-package-dependencies`

Instala as dependências Node.js de um package no diretório de dependências do
ambiente de execução. É uma task de **execução única** (termina em `FINISHED`).

**Parâmetros:**
- `namespace` (string): Identificador único do package (ex: `"@/service-orchestrator.app"`).
- `path` (string): Caminho completo da pasta do package (de onde lê o `package.json`).
- `environmentPath` (string): Caminho do ambiente de execução.
- `EXECUTIONDATA_CONF_DIRNAME_DEPENDENCIES` (string): Nome do diretório de dependências (ex: `".dependencies"`).

**Exemplo no execution params:**
```json
{
  "objectLoaderType": "install-nodejs-package-dependencies",
  "staticParameters": {
    "namespace": "@/service-orchestrator.app",
    "path": "/home/kadisk/EcosystemData/repos/KADISKCorpRepo/.../service-orchestrator.app",
    "environmentPath": "/home/kadisk/EcosystemData/environments/service-orchestrator.app-<hash>",
    "EXECUTIONDATA_CONF_DIRNAME_DEPENDENCIES": ".dependencies"
  }
}
```

---

# `nodejs-package`

Carrega um package Node.js e expõe um **handler** (service object) com `require`,
para que outras tasks consumam o código do package. Normalmente ativa **após** a
instalação das dependências (via `activationRules`).

**Parâmetros:**
- `tag` (string): Identificador único do package (ex: `"@/service-orchestrator.app"`).
- `path` (string): Caminho completo da pasta do package.
- `environmentPath` (string): Caminho do ambiente de execução.
- `EXECUTIONDATA_CONF_DIRNAME_DEPENDENCIES` (string): Nome do diretório de dependências.

O service object exposto oferece `require(srcPath)`, `getSourcePath()`,
`getEnvironmentPath()` e `getNodeModulesPath()` — usados por `service-instance`,
`endpoint-instance` e `command-application` via `linkedParameters`
(`nodejsPackageHandler`).

**Exemplo no execution params:**
```json
{
  "objectLoaderType": "nodejs-package",
  "staticParameters": {
    "tag": "@/service-orchestrator.app",
    "path": "/home/kadisk/EcosystemData/repos/KADISKCorpRepo/.../service-orchestrator.app",
    "environmentPath": "/home/kadisk/EcosystemData/environments/service-orchestrator.app-<hash>",
    "EXECUTIONDATA_CONF_DIRNAME_DEPENDENCIES": ".dependencies"
  },
  "activationRules": {
    "&&": [
      { "property": "params.namespace", "=": "@/service-orchestrator.app" },
      { "property": "status",           "=": "FINISHED" }
    ]
  }
}
```

---

# `command-application`

Instancia uma aplicação de **linha de comando** (CLI). Monta os comandos
(via [`yargs`](https://yargs.js.org/)) a partir dos metadados, executa o comando
solicitado e, ao terminar, encerra o plano de execução (`FINISHED` +
`STOP_ALL_TASKS`).

**Parâmetros:**
- `commands` (array): Metadados dos comandos a registrar (com `command`, `path`, `parameters`, `children`, flags como `isMainCommand`/`isNotStopAllTasks`).
- `commandLineArgs` (array): Argumentos da linha de comando a interpretar.
- `commandParameterNames` (array): Nomes dos parâmetros a repassar ao handler do comando.
- `startupParams` (object): Parâmetros de inicialização repassados ao comando.
- `nodejsPackageHandler` (string → handler): Pacote Node.js que fornece as funções dos comandos (resolvido via `linkedParameters`).

**Exemplo no execution params:**
```json
{
  "objectLoaderType": "command-application",
  "staticParameters": {
    "commandLineArgs": ["list", "installed"],
    "commands": [ { "command": "list", "path": "Commands/List.command", "isMainCommand": true } ],
    "commandParameterNames": ["installDataDirPath"]
  },
  "linkedParameters": {
    "nodejsPackageHandler": "@/repository-manager.cli"
  },
  "agentLinkRules": [ ... ]
}
```

---

# `desktop-window-instance`

Abre uma **janela [Electron](https://www.electronjs.org/)** durante a execução de
um plano. É o object loader que dá suporte aos packages do tipo
[`.desktopapp`](../concepts/package.md): cada entrada da seção `windows` do
`boot.json` vira uma task `desktop-window-instance`. A task permanece `ACTIVE`
enquanto a janela estiver aberta; quando o processo Electron sai (janela fechada),
o loader emite `TERMINATED` para a **própria task**. O encerramento do plano
inteiro (`STOP_ALL_TASKS`) é responsabilidade do `command-application`, não deste
loader.

Suporta dois modos:
- **`loadURL`** (`url`): a janela aponta para uma aplicação web **local** — o
  cenário típico é um `.desktopapp` que sobe o próprio webapp (`services` +
  `endpoints`) e abre a janela em `http://localhost:{port}/`. Via `agentLinkRules`
  (a partir do `bound-param` do serviço), a janela só abre quando o
  `@@/server-service` está `ACTIVE`, e reintenta o load enquanto o webgui ainda
  está compilando.
- **`loadFile`** (`file`, opcionalmente com `dependency`): carrega um HTML
  **local** do package indicado — para conteúdo estático autossuficiente.

> O binário do Electron é uma dependência do package que implementa este loader
> (`desktop-window-instance.lib`), instalada em runtime como qualquer dependência.

**Parâmetros:**
- `url` (string — modo loadURL): URL a carregar (ex.: `http://localhost:8083/`).
- `file` (string — modo loadFile): HTML a carregar, relativo à raiz do package de conteúdo.
- `rootPath` (string — modo loadFile): raiz resolvida do package de conteúdo.
- `title` (string — opcional): Título da janela.
- `width` / `height` (number — opcional): Dimensões iniciais da janela.

**Exemplo no execution params (modo loadURL — `api-designer.desktopapp`):**
```json
{
  "objectLoaderType": "desktop-window-instance",
  "staticParameters": {
    "title": "API Designer",
    "url": "http://localhost:8083/",
    "width": 1280,
    "height": 800
  },
  "linkedParameters": { "serverService": "@@/server-service" },
  "agentLinkRules": [
    {
      "referenceName": "@@/server-service",
      "requirement": {
        "&&": [
          { "property": "params.tag", "=": "@@/server-service" },
          { "property": "status", "=": "ACTIVE" }
        ]
      }
    }
  ]
}
```
