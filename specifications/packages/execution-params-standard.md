# Execution Params Standard

> Especificação do Meta Platform Open Standard. Descreve o formato do
> `execution-params.json` — o **plano de execução** derivado do
> `metadata-hierarchy.json` e consumido pelo Task Executor.
>
> Veja também: [Tipos de Object Loader](../../concepts/tipos-de-object-loader.md),
> [Runtime Environment](../../concepts/runtime-environment.md),
> [Environment Runtime Standard](../environment-runtime-standard.md),
> [Package Executor RPC Standard](../package-executor-rpc-standard.md),
> [Supervisor Socket Standard](../supervisor-socket-standard.md) e
> [Package](../../concepts/package.md) (prefixos de namespace).
>
> Este documento descreve o **comportamento real da implementação de referência
> atual**, não comportamento futuro/desejado.

## Visão geral

O `execution-params.json` é um **array JSON** em que **cada item descreve uma task**
(uma unidade de execução). O conjunto de tasks forma o plano completo para colocar um
package "no ar": instalar dependências, carregar packages Node.js e instanciar a
aplicação, seus serviços e endpoints (ou rodar uma CLI).

Cada task declara **qual object loader** a carrega (`objectLoaderType`), seus
**parâmetros** (`staticParameters`), suas **pré-condições de ativação**
(`activationRules`), as **referências a service objects de outras tasks**
(`linkedParameters` + `agentLinkRules`) e, quando aplicável, suas **tasks filhas**
(`children`).

## Papel no pipeline de runtime

O `execution-params.json` é um artefato intermediário entre o grafo de metadados e a
execução. O fluxo de ponta a ponta:

1. O **Package Executor** resolve o package inicial.
2. Monta o `metadata-hierarchy.json` (grafo do package raiz + dependências resolvidas
   por namespace) e o grava no diretório do ambiente.
3. **Traduz** o grafo em `execution-params.json`
   (`execution-params-generator.lib`, função
   `TranslateMetadataHierarchyForExecutionParams`).
4. **Salva** o resultado em `<environmentPath>/execution-params.json`.
5. **Injeta** dados de isolamento/runtime no plano (ver
   [executionData](#executiondata-e-isolamento-de-ambiente)).
6. O **Task Executor** cria as tasks a partir do plano (`CreateTasks`).
7. Os **object loaders** executam cada task e reportam seu status.
8. O **Supervisor RPC** expõe o estado das tasks via `ListTasks` e `GetTask`.

Artefatos e componentes relacionados:

| Item | Papel |
|------|-------|
| `metadata-hierarchy.json` | Grafo de metadados (entrada da tradução). |
| `execution-params.json` | Plano de execução (saída da tradução; assunto deste documento). |
| `environmentPath` | Diretório do ambiente de execução onde ambos os arquivos são gravados. |
| `.dependencies` | Subdiretório do ambiente onde as dependências Node.js são instaladas (nome em `EXECUTIONDATA_CONF_DIRNAME_DEPENDENCIES`). |
| Task Executor | Consome o plano e orquestra o ciclo de vida das tasks. |
| Package Executor RPC | Contrato gRPC que expõe as tasks (`ListTasks` / `GetTask`). |
| Supervisor Socket | Transporte (socket Unix) sobre o qual o RPC é servido. |

| Pergunta | Resposta |
|----------|----------|
| Quem **gera**? | `execution-params-generator.lib`, invocado pelo Package Executor. |
| A partir de quê? | `metadata-hierarchy.json` (+ `commandLineArgs`, `executableName`, paths do ambiente). |
| Onde é **salvo**? | `<environmentPath>/execution-params.json`. |
| Quem **consome**? | `task-executor.lib`, via `CreateTasks(...)`. |
| Como chega ao supervisor? | RPC `ListTasks` / `GetTask` (mensagens `Task` / `TaskInformation`). |

## Relação com `metadata-hierarchy.json`

O `execution-params.json` é **derivado** do `metadata-hierarchy.json` e fica no mesmo
ambiente de execução. O `metadata-hierarchy.json` descreve **o que** precisa ser
carregado (o grafo de packages); o `execution-params.json` descreve **como e em que
ordem** carregar (as tasks, suas pré-condições e vínculos).

A ordem de geração é determinística:

```
[ ...install-nodejs-package-dependencies (1 por package),
  ...nodejs-package (1 por package),
  application-instance | command-application (no máximo 1) ]
```

## Estrutura raiz

- O arquivo é **sempre um array JSON**.
- **Cada item do array representa uma task** (uma unidade de execução).
- Cada item **precisa ter `objectLoaderType`**. Se ele estiver ausente ou não
  corresponder a um object loader registrado, a criação da task falha (erro
  `"Task Loader was not found"`). Um item vazio (`{}`) também é rejeitado.
- Os **demais campos são opcionais** e muitos **variam conforme o `objectLoaderType`**.
- A **ordem dos itens define a criação e o `taskId`** das tasks, mas **a ativação
  depende das regras** (`activationRules`, `agentLinkRules`, `children`), não da
  posição no array.
- `children` **cria uma subárvore de tasks** (tasks filhas com a task atual como pai).
- O arquivo é **derivado do `metadata-hierarchy.json`** e **consumido pelo Task
  Executor**.

Exemplo mínimo:

```json
[
  {
    "objectLoaderType": "nodejs-package",
    "staticParameters": {
      "tag": "@/example-package",
      "path": "/absolute/path/to/package"
    },
    "activationRules": {
      "&&": [
        {
          "property": "status",
          "=": "FINISHED"
        }
      ]
    }
  }
]
```

> `tag` e `path` são parâmetros reais do loader `nodejs-package`. Este exemplo é
> **simplificado**: o gerador real também inclui `environmentPath` e
> `EXECUTIONDATA_CONF_DIRNAME_DEPENDENCIES` nos `staticParameters`, e usa uma
> `activationRules` com duas condições (`params.namespace = <tag>` **e**
> `status = "FINISHED"`) para aguardar a instalação de dependências do **próprio**
> package. Ver [Exemplos](#exemplos).

## Campos comuns

| Campo | Obrigatório | Tipo | Descrição |
|-------|-------------|------|-----------|
| `objectLoaderType` | **Sim** | string | Identifica o object loader que carrega a task. |
| `staticParameters` | Não | object | Parâmetros imutáveis passados ao loader. **Variam por loader.** |
| `activationRules` | Não | object | Pré-condição de ativação da task (não injeta nada). |
| `linkedParameters` | Não | object | Declara parâmetros resolvidos a partir do service object de outras tasks. |
| `agentLinkRules` | Não | array | Define como encontrar e validar as tasks referenciadas em `linkedParameters`. |
| `children` | Não | array | Define uma subárvore de tasks (tasks filhas). |

Detalhamento:

- **`objectLoaderType`** — obrigatório. Os valores oficiais estão em
  [Object loaders oficiais](#object-loaders-oficiais).
- **`staticParameters`** — varia conforme o `objectLoaderType` (ver a tabela de cada
  loader). Em runtime, recebe automaticamente a chave `executionData`.
- **`activationRules`** — pré-condição de ativação. Usa a
  [gramática de regras](#gramática-de-activationrules-e-requirement).
- **`linkedParameters`** — mapa de nomes locais para referências a service objects de
  outras tasks (ver [linkedParameters e agentLinkRules](#linkedparameters-e-agentlinkrules)).
- **`agentLinkRules`** — define como localizar/validar cada task referenciada e atua
  também como pré-condição.
- **`children`** — subárvore de tasks (ver [children](#children)).

## Campos derivados/injetados em runtime

Os campos a seguir são produzidos pela execução e **não precisam aparecer no arquivo
de entrada**. Estão documentados porque aparecem nas regras (`property`) e no contrato
RPC.

| Campo | Tipo | Origem |
|-------|------|--------|
| `taskId` | number | Atribuído na criação da task. |
| `pTaskId` | number | Id da task pai (presente apenas em tasks filhas). |
| `status` | string | Status corrente da task (um dos nove [status oficiais](#status-de-task)). |
| `params` | object | Fusão (*deep merge*) de `staticParameters` com os `linkedParameters` já resolvidos; criado quando a task chega a `PREPPED_TO_START`. |
| `hasChildTasks` | bool | `true` se a task tem `children`. |
| `executionData` | object | Injetado em `staticParameters` pelo Package Executor (contém `environmentPath`, entre outros). |
| `getServiceObject` | function | Retorno do object loader; consumido por `linkedParameters` de outras tasks. |

> **`params` não é igual a `staticParameters`.** As regras consultam `params.*`
> (ex.: `params.tag`, `params.namespace`), que só existe em runtime a partir de
> `PREPPED_TO_START`.

### `executionData` e isolamento de ambiente

Antes de criar as tasks, o Package Executor injeta em **todas** as tasks (e
recursivamente nas filhas):

- `staticParameters.executionData` — contém, entre outros, o `environmentPath`;
- uma cláusula extra em **cada** `requirement` (de `activationRules` e
  `agentLinkRules`):
  `{ "property": "params.executionData.environmentPath", "=": "<environmentPath>" }`.

Essa cláusula garante que uma regra só case com tasks **do mesmo ambiente de
execução**, isolando execuções simultâneas. O campo não aparece no arquivo gerado pelo
tradutor — é acrescentado em runtime.

## Gramática de `activationRules` e `requirement`

As `activationRules` e os `requirement` de `agentLinkRules` **usam exatamente a mesma
gramática**. A implementação de referência atual suporta:

- **Operador lógico:** apenas `&&` (todas as condições devem ser verdadeiras).
- **Operador de comparação:** apenas `=` (igualdade estrita).
- **`property`:** um caminho usando `.` avaliado **sobre o objeto da task alvo**.

Forma de uma regra:

```json
{
  "&&": [
    {
      "property": "status",
      "=": "FINISHED"
    }
  ]
}
```

Semântica: a regra é convertida em uma *query* e considerada satisfeita se **existe
alguma task** que casa **todas** as condições do `&&`.

### Propriedades acessíveis em `property`

O caminho é resolvido sobre o objeto da task alvo. As propriedades úteis são:

- `status` — o status atual da task.
- `objectLoaderType` — o tipo do loader da task.
- `taskId`, `pTaskId` — identificadores numéricos.
- `params.*` — os parâmetros **resolvidos em runtime** (ex.: `params.tag`,
  `params.namespace`, `params.executionData.environmentPath`).

### `||` não é suportado

> **Nota:** o operador `||` **não é suportado** pela implementação de referência
> atual. O avaliador de regras trata apenas `&&`; uma regra usando `||` será avaliada
> como **não satisfeita**. Caso o `||` venha a ser suportado no futuro, ele deve ser
> implementado no avaliador de regras **antes** de ser documentado como parte do
> standard (ver [Pendências e extensões futuras](#pendências-e-extensões-futuras)).

Da mesma forma, não há outros operadores de comparação (`!=`, `>`, `<`, etc.): apenas
`=`.

### Evitando deadlock

Como não há timeout, uma task cuja regra **nunca** é satisfeita permanece presa em
`AWAITING_PRECONDITIONS` indefinidamente. Em particular, **regras que referenciam
status inexistentes nunca serão satisfeitas**. Garanta que toda condição possa,
eventualmente, ser atingida por alguma task do mesmo ambiente.

## linkedParameters e agentLinkRules

### `linkedParameters`

`linkedParameters` é um **mapa de nomes locais para referências**. Não são "parâmetros
que mudam sozinhos":

- Cada chave é um nome local; o valor é uma **referência** (ex.: `"@/example-service"`,
  `"@@/server-service"`).
- O valor **resolvido em runtime é o service object da task referenciada** (o resultado
  de `getServiceObject()` dela).
- O valor pode ser uma **referência direta** (string) ou uma **estrutura aninhada** de
  referências.
- O resultado resolvido é **mesclado em `params`** (junto com `staticParameters`).

### `agentLinkRules`

`agentLinkRules` define **como encontrar e validar** cada task referenciada. Cada regra
tem:

- `referenceName` — a mesma string usada como referência em `linkedParameters`;
- `requirement` — uma expressão que a task referenciada precisa satisfazer; **segue a
  mesma gramática de `activationRules`** (`&&` + `=`).

Funções dos `agentLinkRules`:

1. **Resolução** — para cada referência de `linkedParameters`, o executor encontra a
   regra cujo `referenceName` é igual à referência, usa o `requirement` para localizar
   a task que o satisfaz e injeta o `getServiceObject()` dela.
2. **Pré-condição (linkage)** — a task só sai de `AWAITING_PRECONDITIONS` quando
   **todos** os `requirement` casam, bloqueando a ativação até a dependência estar
   pronta.

### Diferença entre `activationRules` e `agentLinkRules`

- **`activationRules`** — **apenas pré-condição** (não injeta nada).
- **`agentLinkRules`** — **pré-condição + resolução/injeção** do service object
  referenciado em `linkedParameters`.

> A verificação de linkage só ocorre quando `linkedParameters` **e** `agentLinkRules`
> estão **ambos** presentes. Uma referência em `linkedParameters` sem o `referenceName`
> correspondente em `agentLinkRules` não pode ser resolvida em runtime.

### Exemplo

```json
"linkedParameters": {
  "nodejsPackageHandler": "@/example-service"
},
"agentLinkRules": [
  {
    "referenceName": "@/example-service",
    "requirement": {
      "&&": [
        {
          "property": "params.tag",
          "=": "@/example-service"
        },
        {
          "property": "status",
          "=": "ACTIVE"
        }
      ]
    }
  }
]
```

## Namespaces e resolução de referências

Os prefixos seguem a definição canônica em [Package](../../concepts/package.md):

- `@/` — referência a um **package por namespace** no conjunto de repositórios
  instalados (ex.: `@/server-manager.service`).
- `@@/` — referência a uma **instância/serviço dentro do mesmo contexto de execução**
  (ex.: `@@/server-service`). É o identificador (`tag`) de uma task no mesmo plano.
- `@//` — referência **interna do próprio package**, usada em **metadados/boot**.
  Normalmente **não aparece** no `execution-params.json` resolvido; é listado aqui
  apenas para evitar confusão.

Onde as referências podem aparecer:

- `linkedParameters` (chaves/valores de referência);
- `agentLinkRules.referenceName`;
- `activationRules` (como valor de `=`);
- `agentLinkRules.requirement` (como valor de `=`).

Como são resolvidas: as referências de `linkedParameters` / `agentLinkRules.referenceName`
são comparadas, via `requirement`, contra propriedades das tasks (tipicamente
`params.tag` ou `params.namespace`). A resolução é **por identidade da task**, nunca
por posição na árvore nem por caminho de arquivo.

## children

- O **Task Executor suporta `children`** como uma **subárvore de tasks**: as filhas são
  criadas recursivamente, tendo a task atual como pai (`pTaskId`).
- Na **implementação de referência**, `children` é gerado **principalmente para
  `application-instance`**.
- Os filhos costumam ser `service-instance` e `endpoint-instance`.
- Uma **task pai pode depender do estado dos filhos**: na implementação de referência,
  uma task com filhas só ativa quando **todas as filhas estão `ACTIVE`**.
- `children` **não é uma limitação absoluta do formato**, mas uma **convenção da
  implementação de referência** — o executor permite filhos em qualquer task.

## Object loaders oficiais

Os sete object loaders oficiais (registrados pelo Package Executor e implementados em
`EssentialTaskLoaders.layer`). Para descrição detalhada, ver
[Tipos de Object Loader](../../concepts/tipos-de-object-loader.md).

| `objectLoaderType` | Papel | `staticParameters` principais | `linkedParameters` | `agentLinkRules` | `children` |
|--------------------|-------|-------------------------------|--------------------|------------------|-----------|
| `install-nodejs-package-dependencies` | Instala as dependências Node.js do package (execução única). | `namespace`, `path`, `environmentPath`, `EXECUTIONDATA_CONF_DIRNAME_DEPENDENCIES` | — | — | não |
| `nodejs-package` | Carrega o package Node.js e expõe um handler (service object). | `tag`, `path`, `environmentPath`, `EXECUTIONDATA_CONF_DIRNAME_DEPENDENCIES` | — | — | não |
| `application-instance` | Instancia uma aplicação completa. | `startupParams`, `namespace`, `rootPath` | — | — | **sim** |
| `service-instance` | Instancia um serviço dentro de uma aplicação. | `tag`, `path`, `serviceParameterNames` (+ params) | `nodejsPackageHandler` (+ bound params) | sim | não¹ |
| `endpoint-instance` | Monta um endpoint HTTP. | `url`, `type` (+ params) | `nodejsPackageHandler` (+ bound params) | sim | não¹ |
| `command-application` | Instancia uma aplicação de linha de comando (CLI). | `startupParams`, `namespace`, `rootPath`, `commands`, `executableName`, `commandLineArgs`, `commandParameterNames` | `nodejsPackageHandler` (+ bound params) | sim | não |
| `desktop-window-instance` | Abre uma janela Electron. Modo primário: `loadURL` para uma app web local que sobe junto (backend + webgui). Modo alternativo: `loadFile` para HTML estático. | `url` (loadURL) **ou** `file`/`rootPath` (loadFile) + `namespace` (+ `title`, `width`, `height`) | `serverService` (loadURL) | sim (loadURL) | não |

¹ Gerados como **filhos** de `application-instance`, não como itens do array raiz.

Observações:

- **`application-instance`** normalmente cria `children` (seus `service-instance` e
  `endpoint-instance`).
- **`service-instance`** e **`endpoint-instance`** aparecem, na implementação de
  referência, como **filhos** de `application-instance`.
- **`endpoint-instance`** pode representar um **`controller`** ou uma **interface web**
  (`web-graphic-user-interface`): o campo `staticParameters.type` seleciona o
  comportamento (confirmado no loader de referência).
- **`command-application`** é uma execução de **CLI / vida curta**: ao concluir o
  comando, emite `FINISHED` **e encerra todo o plano de execução**.
- A escolha entre `application-instance` e `command-application` é feita pelo gerador:
  com `boot`, usa `application-instance`; com executáveis **e** um `executableName`
  solicitado, usa `command-application`.
- **`desktop-window-instance`** dá suporte aos packages `.desktopapp`: cada entrada
  da seção `windows` do `boot.json` vira uma task. No **modo primário** (`loadURL`), a
  janela aponta para uma app web local que sobe junto no mesmo plano (os `services` e
  `endpoints` que sobem o backend HTTP e o webgui, compilado em runtime); o parâmetro
  `url` recebe um valor inteiro via template (ex.: `"url": "{{windowUrl}}"`, onde
  `windowUrl` é uma startup-param como `http://localhost:8083/`) — interpolação
  embutida na string **não** é suportada. A janela espera o serviço `@@/server-service`
  ficar `ACTIVE` (via `linkedParameters.serverService` + `agentLinkRules`, gerados a
  partir do `bound-param` `serverService`) e reintenta o load enquanto o webgui compila.
  No **modo alternativo** (`loadFile`), a janela carrega HTML estático do disco via
  `BrowserWindow.loadFile` (sem servidor HTTP), podendo o conteúdo vir de outro package
  (via `dependency` na janela), resolvido em `rootPath`. Em ambos os modos, a task
  permanece `ACTIVE` enquanto a janela estiver aberta e **encerra o plano de execução**
  ao ser fechada.

## Status de task

Os status oficiais estão definidos em
[`proto/package_executor_rpc.proto`](../../proto/package_executor_rpc.proto)
(enum `TaskStatus`) e coincidem com os usados em runtime pelo Task Executor:

| Status | Tipo | Significado |
|--------|------|-------------|
| `AWAITING_PRECONDITIONS` | transitório | Estado inicial; aguardando suas pré-condições. |
| `PRECONDITIONS_COMPLETED` | transitório | Pré-condições satisfeitas. |
| `PREPPED_TO_START` | transitório | `params` e service object montados; pronta para iniciar. |
| `STARTING` | transitório | O loader está iniciando a task. |
| `ACTIVE` | **estável (saudável)** | Serviço/instância ativo e no ar. |
| `STOPPING` | transitório | Encerrando. |
| `FINISHED` | **final (saudável)** | Execução única concluída com sucesso (ex.: instalação, CLI). |
| `FAILURE` | **final** | A task falhou. |
| `TERMINATED` | **final** | A task foi encerrada. |

- **Transitórios:** `AWAITING_PRECONDITIONS`, `PRECONDITIONS_COMPLETED`,
  `PREPPED_TO_START`, `STARTING`, `STOPPING`.
- **Finais:** `FINISHED`, `FAILURE`, `TERMINATED`.
- **Execução saudável:** todas as tasks em `ACTIVE` **ou** `FINISHED` e **nenhuma** em
  `FAILURE`. `ACTIVE` é o estado estável de serviços de longa duração; `FINISHED` é o
  estado final de tasks de execução única.
- **Falha:** `FAILURE` representa falha da task.

> **Atenção:** `PENDING` e `FAILED` **não existem**. Regras que comparem `status`
> contra valores inexistentes **nunca serão satisfeitas** e travam a task em
> `AWAITING_PRECONDITIONS`. O equivalente correto de "falhou" é `FAILURE`; o de
> "encerrada" é `TERMINATED`.

Ciclo de vida (resumo):

```
AWAITING_PRECONDITIONS → PRECONDITIONS_COMPLETED → PREPPED_TO_START → STARTING → ACTIVE | FINISHED
                                                                                  ↓
                                                              (parada)  STOPPING → TERMINATED
                                                              (erro)             → FAILURE
```

## Relação com o Package Executor RPC

As tasks criadas a partir do `execution-params.json` são expostas pelo contrato gRPC
definido em [Package Executor RPC Standard](../package-executor-rpc-standard.md). Os
RPCs `ListTasks` (mensagem `Task`) e `GetTask` (mensagem `TaskInformation`) reportam o
estado das tasks; o enum `TaskStatus` é o mesmo da seção [Status de task](#status-de-task).

A mensagem `TaskInformation` expõe:

- `taskId`
- `pTaskId`
- `objectLoaderType`
- `status`
- `staticParameters`
- `linkedParameters`
- `agentLinkRules`
- `activationRules`
- `hasChildTasks`

Ou seja, o que o supervisor reporta espelha diretamente os campos do
`execution-params.json` mais o status corrente e a relação pai/filho.

## Exemplos

Os exemplos a seguir são **simplificados** e derivados da forma produzida pelo gerador
(`execution-params-generator.lib`) — **não** são a saída exata de uma execução real. O
campo `executionData` e a cláusula de isolamento (injetados em runtime) estão omitidos
para clareza. Todo JSON é válido e não contém comentários.

### `install-nodejs-package-dependencies` (exemplo simplificado)

```json
{
  "objectLoaderType": "install-nodejs-package-dependencies",
  "staticParameters": {
    "namespace": "@/example.app",
    "path": "/absolute/path/to/example.app",
    "environmentPath": "/absolute/path/to/environments/example.app-hash",
    "EXECUTIONDATA_CONF_DIRNAME_DEPENDENCIES": ".dependencies"
  }
}
```

### `nodejs-package` (exemplo simplificado)

```json
{
  "objectLoaderType": "nodejs-package",
  "staticParameters": {
    "tag": "@/example.app",
    "path": "/absolute/path/to/example.app",
    "environmentPath": "/absolute/path/to/environments/example.app-hash",
    "EXECUTIONDATA_CONF_DIRNAME_DEPENDENCIES": ".dependencies"
  },
  "activationRules": {
    "&&": [
      { "property": "params.namespace", "=": "@/example.app" },
      { "property": "status",           "=": "FINISHED" }
    ]
  }
}
```

### `application-instance` com `children` (exemplo simplificado)

```json
{
  "objectLoaderType": "application-instance",
  "staticParameters": {
    "startupParams": { "serverName": "ExampleAppInstance" },
    "namespace": "@/example.app",
    "rootPath": "/absolute/path/to/example.app"
  },
  "children": [
    {
      "objectLoaderType": "service-instance",
      "staticParameters": {
        "tag": "@@/example-service",
        "path": "Services/Example.service",
        "serviceParameterNames": ["name"]
      },
      "linkedParameters": { "nodejsPackageHandler": "@/example.app" },
      "agentLinkRules": [
        {
          "referenceName": "@/example.app",
          "requirement": {
            "&&": [
              { "property": "params.tag", "=": "@/example.app" },
              { "property": "status",     "=": "ACTIVE" }
            ]
          }
        }
      ]
    }
  ]
}
```

### `service-instance` (exemplo simplificado)

```json
{
  "objectLoaderType": "service-instance",
  "staticParameters": {
    "tag": "@@/server-service",
    "name": "ServiceOrchestratorAppInstance",
    "path": "Services/HTTPServer.service",
    "serviceParameterNames": ["name", "port", "authenticationService"]
  },
  "linkedParameters": {
    "nodejsPackageHandler": "@/server-manager.service"
  },
  "agentLinkRules": [
    {
      "referenceName": "@/server-manager.service",
      "requirement": {
        "&&": [
          { "property": "params.tag", "=": "@/server-manager.service" },
          { "property": "status",     "=": "ACTIVE" }
        ]
      }
    }
  ]
}
```

### `endpoint-instance` (exemplo simplificado)

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
    "controllerParams": { "httpServerService": "@@/server-service" },
    "serverService": "@@/server-service"
  },
  "agentLinkRules": [
    {
      "referenceName": "@@/server-service",
      "requirement": {
        "&&": [
          { "property": "params.tag", "=": "@@/server-service" },
          { "property": "status",     "=": "ACTIVE" }
        ]
      }
    }
  ]
}
```

> `type` aceita `"controller"` ou `"web-graphic-user-interface"`.

### `command-application` (exemplo simplificado)

```json
{
  "objectLoaderType": "command-application",
  "staticParameters": {
    "namespace": "@/repository-manager.cli",
    "rootPath": "/absolute/path/to/repository-manager.cli",
    "commandLineArgs": ["list", "installed"],
    "commands": [
      { "command": "list", "path": "Commands/List.command", "isMainCommand": true }
    ],
    "commandParameterNames": ["installDataDirPath"]
  },
  "linkedParameters": {
    "nodejsPackageHandler": "@/repository-manager.cli"
  },
  "agentLinkRules": [
    {
      "referenceName": "@/repository-manager.cli",
      "requirement": {
        "&&": [
          { "property": "params.tag", "=": "@/repository-manager.cli" },
          { "property": "status",     "=": "ACTIVE" }
        ]
      }
    }
  ]
}
```

### `desktop-window-instance` (exemplo simplificado)

Modo primário (`loadURL`) — o `api-designer.desktopapp` roda a mesma app web do
`api-designer.webapp` (mesmos `services`/`endpoints`) dentro de uma janela Electron,
na porta `8083`, esperando o `@@/server-service` ficar `ACTIVE`:

```json
{
  "objectLoaderType": "desktop-window-instance",
  "staticParameters": { "title": "API Designer", "url": "http://localhost:8083/", "width": 1280, "height": 800 },
  "linkedParameters": { "serverService": "@@/server-service" },
  "agentLinkRules": [ { "referenceName": "@@/server-service", "requirement": { "&&": [ { "property": "params.tag", "=": "@@/server-service" }, { "property": "status", "=": "ACTIVE" } ] } } ]
}
```

> No modo `loadURL`, `url` é um valor inteiro (template `{{windowUrl}}` de uma
> startup-param); interpolação embutida na string não é suportada.

Modo alternativo (`loadFile`) — carrega HTML estático do disco (sem servidor HTTP):

```json
{
  "objectLoaderType": "desktop-window-instance",
  "staticParameters": {
    "title": "Static App",
    "file": "dist/index.html",
    "width": 1280,
    "height": 800,
    "namespace": "@/static-app.desktopapp",
    "rootPath": "/absolute/path/to/static-app.webgui"
  }
}
```

> `file` é relativo ao package que fornece o conteúdo; `rootPath` é a raiz resolvida
> desse package (o próprio `.desktopapp` ou o package indicado por `dependency` na
> janela do `boot.json`).

### Exemplo inválido

O exemplo abaixo é **inválido na implementação atual** por dois motivos:

```json
{
  "objectLoaderType": "nodejs-package",
  "activationRules": {
    "||": [
      { "property": "status", "=": "FAILED" }
    ]
  }
}
```

- Usa o operador lógico `||`, que **não é suportado** pelo avaliador de regras — a
  regra será avaliada como não satisfeita e a task nunca ativará.
- Compara `status` com `FAILED`, que **não é um status oficial** (o correto seria
  `FAILURE`) — a condição nunca casaria mesmo com `&&`.

## Validação mínima

Um `execution-params.json` válido deve:

- ser um **array JSON**;
- ter `objectLoaderType` em **cada item**;
- usar **apenas object loaders registrados** (oficiais);
- usar **apenas status oficiais** nas regras;
- usar **apenas `&&` e `=`** nas regras;
- não ter referências em `linkedParameters` **sem** o `agentLinkRules` correspondente;
- evitar `activationRules`/`requirement` que nunca podem ser satisfeitos (deadlock);
- preservar referências **resolvíveis** (a task referenciada precisa existir no plano).

### JSON Schema futuro

Um **JSON Schema formal** para o `execution-params.json` ainda **deve ser criado**.
Quando existir, deverá modelar: o array; `objectLoaderType` como `enum` dos loaders
oficiais; a forma `{ "&&": [ { property, "=" } ] }` das regras; e a estrutura recursiva
de `children`. Ver [Pendências e extensões futuras](#pendências-e-extensões-futuras).

## Boas práticas

- Preferir referências por **namespace/tag estável** (`@/`, `@@/`), nunca por caminho
  de arquivo ou posição no array.
- Evitar regras que dependam de **status inexistentes**.
- Evitar **dependências circulares** entre tasks (risco de deadlock).
- Validar `linkedParameters` **junto com** `agentLinkRules` (toda referência precisa de
  um `referenceName` correspondente).
- Gerar **exemplos reais** a partir do Package Executor sempre que possível, em vez de
  escrever planos à mão.
- **Não usar operadores não implementados** (`||` e comparadores além de `=`).
- Manter o documento **sincronizado com os task loaders oficiais**.

## Pendências e extensões futuras

- **Suporte a `||`** — não suportado hoje; só deve ser documentado como parte do
  standard **após** ser implementado no avaliador de regras.
- **JSON Schema formal** do `execution-params.json`.
- **Exemplos reais versionados** de `execution-params.json` (saída de execuções reais).
- **Documentação mais detalhada por task loader** (parâmetros completos de cada loader).
- **Validação automatizada** entre os object loaders registrados e esta especificação.
</content>
