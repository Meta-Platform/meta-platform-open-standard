# Package Metadata Standard

Define os arquivos da pasta `metadata/` de um [Package](../concepts/package.md) —
o que torna o package compreensível para a plataforma **sem executá-lo**.

O nome da pasta de metadados é configurável por
[`ecosystem-defaults.json`](./metadados/ecosystem-defaults.json)
(`PKG_CONF_DIRNAME_METADATA`, padrão `metadata`).

## `package.json` (metadata) — obrigatório

Declara o **namespace** do package. Único metadado obrigatório de qualquer
package.

```json
{ "namespace": "@/repository-manager.cli" }
```

Prefixos de namespace: `@/` (package por namespace no conjunto de repositórios
instalados — declarado dentro de um repositório, resolvido globalmente no
`EcosystemData`), `@@/` (instância/serviço no mesmo contexto de execução), `@//`
(referência interna de boot). Ver
[Dependency Resolution Standard](./dependency-resolution-standard.md).

> Não confundir com o `package.json` da **raiz** do package (identidade e
> dependências NPM). Este fica em `metadata/`.

## `boot.json` — packages executáveis

Descreve o que o package **expõe** ao ser carregado. Conteúdo varia conforme o
tipo. A especificação completa está em
[Executable Manifest Standard](./executable-manifest-standard.md). Forma para CLI:

```json
{
    "executables": [
        {
            "dependency": "@//command-group",
            "executableName": "repo",
            "bound-params": {
                "ecosystemInstallUtilitiesLib": "@/ecosystem-install-utilities.lib",
                "printDataLogLib": "@/print-data-log.lib"
            }
        }
    ]
}
```

Pacotes de aplicação/web (`.app`, `.webapp`, `.webgui`, `.webservice`) usam um
`boot.json` com `params`, `services` e `endpoints` (ver
[Executable Manifest Standard](./executable-manifest-standard.md)).

Pacotes desktop (`.desktopapp`) usam um `boot.json` com a seção `windows`, que
declara as janelas [Electron](https://www.electronjs.org/) a abrir. Tipicamente
o `.desktopapp` **combina** `services`/`endpoints` (o backend/web local, como um
webapp) com `windows` (a janela que faz `loadURL` desse servidor local); também
suporta `loadFile` de HTML estático. Ver
[Executable Manifest Standard](./executable-manifest-standard.md).

## `command-group.json` — packages `.cli`

A **árvore de comandos** da CLI.

| Campo | Função |
|-------|--------|
| `commandName` | Nome interno do comando. |
| `path` | Handler em `src/` (sem `.js`), ex.: `Commands/Install.command`. |
| `command` | Assinatura no estilo yargs (`install [repositoryNamespace] [sourceType]`). |
| `description` | Texto do `--help`. |
| `parameters` | `paramType` (`positional`/`option`), `valueType` (`string`/`array`/…), `key`, `describe`. |
| `parametersToLoad` | Quais `bound-params` são injetados no comando. |
| `children` | (opcional) subcomandos. |

## `services.json` — packages `.service`

Lista os serviços expostos por um package de serviços. Cada entrada:

```json
[
    {
        "namespace": "HTTPServerService",
        "path": "Services/HTTPServer.service",
        "bound-params": ["?middlewareService"],
        "params": ["name", "port"]
    }
]
```

- `namespace` — nome do serviço (referenciável como
  `@/<pacote>.service/services/<namespace>`).
- `path` — arquivo do serviço em `src/`.
- `bound-params` — dependências por namespace (`?` = opcional).
- `params` — parâmetros estáticos esperados.

## `endpoint-group.json` — packages web

Presente em `.webgui`/`.webservice` e nos `.app` que expõem endpoints; descreve o
grupo de endpoints (controllers/APIs/interface) que o package monta sobre um
`@@/server-service`. **Não** aparece em `.webapp` (o `.webapp` compõe seus
serviços/endpoints pelo `boot.json`).

## `startup-params.json` — opcional

Valores de inicialização **específicos do package**, entregues aos handlers como
`startupParams` (ex.: `{ "installDataDirPath": "~/EcosystemData" }`).

> **Contrato:** o `startup-params.json` guarda apenas o que é específico do
> package (`port`, `serverName`, `socket`, caminhos próprios, ...). As variáveis
> de configuração do ecossistema definidas em `ecosystem-defaults.json`
> (`REPOS_CONF_*`, `PKG_CONF_*`, `ECOSYSTEMDATA_CONF_*`, `EXECUTIONDATA_CONF_*`)
> **não** devem ser copiadas aqui: o executor (`pkg-exec`) injeta o
> `ecosystem-defaults` como base do `startupParams` na execução, então os
> `{{VAR}}` referenciados no `boot.json` resolvem por **herança**. Copiar essas
> variáveis dessincroniza o package do ecossistema quando um default muda.

## `startup-params-schema.json` — opcional

Presente na maioria dos packages de aplicação/web (`.webapp`, `.webgui`,
`.webservice`, `.app`, `.desktopapp`), acompanha o `startup-params.json`. Descreve
o schema dos parâmetros de inicialização (reservado para tooling; sem consumidor no
runtime de referência atual).

## Resumo por tipo

| Arquivo | `.lib` | `.cli` | `.service` | `.app`/`.webapp`/`.webgui`/`.webservice` | `.desktopapp` |
|---------|:------:|:------:|:----------:|:----------------------------------------:|:-------------:|
| `package.json` (namespace) | ✅ | ✅ | ✅ | ✅ | ✅ |
| `boot.json` | — | ✅ | — | ✅ (params/services/endpoints) | ✅ (windows) |
| `command-group.json` | — | ✅ | — | — | — |
| `services.json` | — | — | ✅ | — | — |
| `endpoint-group.json` | — | — | — | ✅ (`.webgui`/`.webservice`/`.app` com endpoints; **não** `.webapp`) | — |
| `startup-params.json` | — | opc. | opc. | opc. | opc. |
| `startup-params-schema.json` | — | — | — | opc. (maioria) | opc. |
