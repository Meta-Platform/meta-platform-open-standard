
# Tipos de Object Loader


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
- `path` (string): Caminho relativo ao arquivo do serviço
- `serviceParameterNames` (array): Lista de nomes de parâmetros esperados
- `serviceStorageFilePath` (string - opcional): Caminho do banco de dados
- `instanceDataDirPath` (string - opcional): Diretório de dados da instância
- `nodejsPackageHandler` (string): Referência ao pacote Node.js

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
