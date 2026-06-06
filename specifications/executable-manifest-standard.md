# Executable Manifest Standard

Define o `metadata/boot.json` — o **manifesto** que descreve o que um
[Package](../concepts/package.md) expõe ao ser carregado e como gera um
[Executable](../concepts/executable.md).

Há duas formas principais de `boot.json`, conforme o package gere uma **CLI** ou
uma **aplicação/serviço web**.

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

## Referências cruzadas

- A árvore de comandos da CLI: [Package Metadata Standard](./package-metadata-standard.md#command-groupjson--packages-cli).
- Como o executável é instalado e iniciado: [Executable](../concepts/executable.md).
- Como o manifesto vira plano de execução: [Environment Runtime Standard](./environment-runtime-standard.md)
  e [Execution Params Standard](./packages/execution-params-standard.md).
