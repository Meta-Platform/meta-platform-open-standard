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

Prefixos de namespace: `@/` (package no repositório), `@@/` (instância/serviço no
mesmo contexto de execução), `@//` (referência interna de boot). Ver
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

Presente em `.webgui`/`.webservice`/`.webapp`; descreve o grupo de endpoints
(controllers/APIs/interface) que o package monta sobre um `@@/server-service`.

## `startup-params.json` — opcional

Valores de inicialização específicos do package, entregues aos handlers como
`startupParams` (ex.: `{ "installDataDirPath": "~/EcosystemData" }`).

## Resumo por tipo

| Arquivo | `.lib` | `.cli` | `.service` | `.app`/`.webapp`/`.webgui`/`.webservice` |
|---------|:------:|:------:|:----------:|:----------------------------------------:|
| `package.json` (namespace) | ✅ | ✅ | ✅ | ✅ |
| `boot.json` | — | ✅ | ✅ (services) | ✅ (params/services/endpoints) |
| `command-group.json` | — | ✅ | — | — |
| `services.json` | — | — | ✅ | — |
| `endpoint-group.json` | — | — | — | ✅ (web) |
| `startup-params.json` | — | opc. | opc. | opc. |
