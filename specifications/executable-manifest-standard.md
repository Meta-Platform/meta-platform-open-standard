# Executable Manifest Standard

Define o `metadata/boot.json` — o **manifesto** que descreve o que um
[Package](../concepts/package.md) expõe ao ser carregado e como gera um
[Executable](../concepts/executable.md).

Há três formas principais de `boot.json`, conforme o package gere uma **CLI**,
uma **aplicação/serviço web** ou uma **aplicação desktop**.

## Forma 1 — CLI (`.cli`)

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

| Campo | Função |
|-------|--------|
| `executables[]` | Lista de executáveis expostos. |
| `dependency` | O que constrói o executável. `@//command-group` indica que ele é montado a partir do `command-group.json` do próprio package. |
| `executableName` | Nome do comando no terminal (ex.: `repo`). |
| `bound-params` | Dependências resolvidas por **namespace**, expostas ao executável sob nomes amigáveis (`...Lib`, `...Service`). |
| `params` | (opcional) parâmetros estáticos, com suporte a *templating* `{{...}}`. |

## Forma 2 — Aplicação / Web (`.app`, `.webapp`, `.webgui`, `.webservice`)

Declara `params` (entradas do package), `services` (serviços a instanciar) e
`endpoints` (endpoints/interface a montar). Exemplo reduzido (real, do
`ecosystem-control-panel.webapp`):

```json
{
    "params": ["port", "serverName", "installDataDirPath", "..."],
    "services": [
        {
            "namespace": "@@/server-service",
            "dependency": "@/server-manager.service/services/HTTPServerService",
            "params": { "name": "{{serverName}}", "port": "{{port}}" }
        }
    ],
    "endpoints": [
        {
            "dependency": "@/ecosystem-control-panel.webservice/endpoint-group",
            "bound-params": { "serverService": "@@/server-service" }
        }
    ]
}
```

| Campo | Função |
|-------|--------|
| `params[]` | Nomes dos parâmetros de inicialização que o package aceita. |
| `services[]` | Serviços instanciados. `namespace` (ex.: `@@/server-service`) é a tag local; `dependency` aponta o serviço de origem (`@/<pkg>.service/services/<Service>`); `bound-params`/`params` ligam dependências e valores. |
| `endpoints[]` | Endpoints/interfaces montados sobre os serviços, via `dependency` para um `endpoint-group` (`@//endpoint-group` próprio ou de outro package). |

### Templating `{{...}}`

Valores como `"{{serverName}}"` em `params` são substituídos pelos
`startup-params`/`params` resolvidos no momento da execução.

## Forma 3 — Aplicação desktop (`.desktopapp`)

Declara uma seção `windows`: cada entrada vira uma janela
[Electron](https://www.electronjs.org/) instanciada pelo object loader
[`desktop-window-instance`](../concepts/tipos-de-object-loader.md#desktop-window-instance).
O caso típico **combina a Forma 2** (`services` + `endpoints` que sobem um backend
HTTP local e o webgui) com `windows` — a janela apenas encapsula essa aplicação
web. Exemplo real (`api-designer.desktopapp`, reduzido):

```json
{
    "params": ["apisDirPath", "port", "serverManagerUrl", "windowUrl", "serverName", "RT_ENV_GENERATED_DIR_NAME"],
    "services": [
        { "namespace": "@@/server-service", "dependency": "@/server-manager.service/services/HTTPServerService", "params": { "name": "{{serverName}}", "port": "{{port}}" } }
    ],
    "endpoints": [
        { "dependency": "@/server-manager.webservice/endpoint-group", "bound-params": { "serverService": "@@/server-service" } },
        { "dependency": "@/api-designer.webservice/endpoint-group", "params": { "apisDirPath": "{{apisDirPath}}" }, "bound-params": { "serverService": "@@/server-service" } },
        { "dependency": "@/api-designer.webgui/endpoint-group", "params": { "serverManagerUrl": "{{serverManagerUrl}}", "serverName": "{{serverName}}", "RT_ENV_GENERATED_DIR_NAME": "{{RT_ENV_GENERATED_DIR_NAME}}" }, "bound-params": { "serverService": "@@/server-service" } }
    ],
    "windows": [
        { "title": "API Designer", "url": "{{windowUrl}}", "width": 1280, "height": 800, "bound-params": { "serverService": "@@/server-service" } }
    ]
}
```

| Campo | Função |
|-------|--------|
| `windows[]` | Lista de janelas Electron a abrir. |
| `title` | (opcional) Título da janela. |
| `url` | (modo **loadURL**) URL a carregar — normalmente a aplicação web local que sobe junto (ex.: `{{windowUrl}}` → `http://localhost:{port}/`). Templates são **valor inteiro** (`"{{windowUrl}}"`), como nos demais params. |
| `file` | (modo **loadFile**) HTML a carregar, relativo à raiz do package de conteúdo — para conteúdo estático. |
| `dependency` | (opcional, modo loadFile) Namespace do package que fornece o `file`. Default: o próprio `.desktopapp`. |
| `bound-params` | (opcional) Serviços de instância a aguardar; geram `agentLinkRules` para a janela só abrir quando o serviço (ex.: `@@/server-service`) estiver `ACTIVE`. |
| `width` / `height` | (opcional) Dimensões iniciais da janela. |

> A janela é `loadURL` **ou** `loadFile`. No modo `loadURL`, o `.desktopapp` sobe
> um backend HTTP local (Forma 2) e a janela aponta para ele — é como o
> `api-designer.desktopapp` funciona. No modo `loadFile`, a janela carrega um HTML
> do disco, sem backend.

## Referências cruzadas

- A árvore de comandos da CLI: [Package Metadata Standard](./package-metadata-standard.md#command-groupjson--packages-cli).
- Como o executável é instalado e iniciado: [Executable](../concepts/executable.md).
- Como o manifesto vira plano de execução: [Environment Runtime Standard](./environment-runtime-standard.md)
  e [Execution Params Standard](./packages/execution-params-standard.md).
