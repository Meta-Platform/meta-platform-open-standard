# Repository Metadata Standard

Define os **metadados de um [Repository](../concepts/repository.md)** e como ele
publica seus [executáveis](../concepts/executable.md) para o ecossistema.

Para a **estrutura de itens** do repositório (`Module → Layer → Group → Package`),
veja [Module / Layer / Group](../concepts/module-layer-group.md).

## `metadata/applications.json`

Lista as aplicações/executáveis que o repositório publica. Cada entrada associa
um [package](../concepts/package.md) a um executável instalável e a um socket de
supervisão:

```json
[
    {
        "appType": "CLI",
        "executable": "repo",
        "packageNamespace": "Main.Module/Application.layer/repository-manager.cli",
        "supervisorSocketFileName": "repo.sock"
    }
]
```

| Campo | Descrição |
|-------|-----------|
| `appType` | Tipo do executável. Valores observados nos repositórios oficiais: `CLI`, `APP`, `DESKTOP` (aplicação desktop Electron, `.desktopapp`). |
| `executable` | Nome do comando instalado em `EcosystemData/executables/` (precisa estar no `PATH`). |
| `packageNamespace` | Caminho do package a partir do Module (`Module/Layer/[Group/]Package`). Pode atravessar um `*.group`. |
| `supervisorSocketFileName` | Nome do [supervisor socket](./supervisor-socket-standard.md) criado em `EcosystemData/supervisor-sockets/`. |

> A tripla `executable` ↔ `packageNamespace` ↔ `supervisorSocketFileName` é o que
> liga **o comando que o usuário roda** → **o package que será executado** → **o
> socket pelo qual o processo é supervisionado**.

## `metadata/repository.json`

Declara a **identidade e as capacidades** que o repositório publica para o
ecossistema: seu namespace, de quais repositórios ele depende e quais tipos de
package ele habilita.

```json
{
    "namespace": "EcosystemCoreRepo",
    "dependencies": ["EssentialRepo"],
    "supportedPackageTypes": ["webapp", "webgui", "webservice"]
}
```

| Campo | Descrição |
|-------|-----------|
| `namespace` | Namespace do repositório (a mesma chave usada em `sources.json`/`repositories.json`). Ex.: `EssentialRepo`, `EcosystemCoreRepo`, `PlatformApplicationsRepo`. |
| `dependencies` | Lista de namespaces de repositórios dos quais este depende. Ex.: `EcosystemCoreRepo` depende de `EssentialRepo`; `PlatformApplicationsRepo` de `EssentialRepo` + `EcosystemCoreRepo`. |
| `supportedPackageTypes` | Tipos de package que este repositório **habilita** no ecossistema. `EssentialRepo` habilita `app/cli/service/lib`; `EcosystemCoreRepo` acrescenta `webapp/webgui/webservice`; `PlatformApplicationsRepo` acrescenta `desktopapp`. |

## `metadata/taskloaders.json`

Declara os [object loaders](../concepts/tipos-de-object-loader.md) (packages
`.taskLoader`) que o repositório **fornece**, com as dependências npm que cada um
exige em runtime:

```json
{
    "taskLoaders": [
        {
            "objectLoaderType": "endpoint-instance",
            "package": "@/endpoint-instance.taskLoader",
            "entry": "src/EndpointInstance.taskLoader",
            "npmDependencies": { "webpack": "^5.102.1", "html-webpack-plugin": "^5.6.4", "colors": "^1.4.0" }
        }
    ]
}
```

| Campo | Descrição |
|-------|-----------|
| `objectLoaderType` | A string que identifica o loader no `execution-params.json` (ex.: `endpoint-instance`). |
| `package` | Namespace do package `.taskLoader` que implementa o loader (ex.: `@/endpoint-instance.taskLoader`). |
| `entry` | Caminho do módulo de entrada dentro do package (ex.: `src/EndpointInstance.taskLoader`). |
| `npmDependencies` | Pacotes npm que o loader precisa; a implementação combina os `npmDependencies` de todos os taskloaders instalados no diretório de dependências do ecossistema. |

> Repositórios sem loaders próprios declaram `"taskLoaders": []`. Os 7 loaders da
> implementação de referência ficam no `EssentialRepo`
> (`Taskloaders.Module/Loaders.layer/*.taskLoader`).

## Fontes (`sources.json`) e instalação

Um repositório é descoberto/instalado a partir de uma **fonte** registrada no
`sources.json` do ecossistema:

| `sourceType` | Origem | Campos persistidos | Argumentos da CLI `repo register source` |
|--------------|--------|--------------------|-------------------------------------------|
| `LOCAL_FS` | Sistema de arquivos local | `path` | `--localPath` |
| `GITHUB_RELEASE` | Release do GitHub | `repositoryOwner`, `repositoryName` | `--repoOwner`, `--repoName` |
| `GOOGLE_DRIVE` | `.tar.gz` no Google Drive | `fileId` | `--fileId` |

> Os nomes dos **campos persistidos** no `sources.json` (`path`,
> `repositoryOwner`/`repositoryName`, `fileId`) diferem dos **argumentos da CLI**
> (`--localPath`, `--repoOwner`/`--repoName`): a CLI converte os argumentos para os
> campos ao registrar a fonte.

A gestão é feita pela CLI `repo` (`repository-manager.cli`) e pelo
[Setup Wizard](https://github.com/Meta-Platform/meta-platform-setup-wizard-command-line)
via [Installation Profiles](./ecosystem-installation-profile-standard.md). Os
arquivos `sources.json` e `repositories.json` do ecossistema instalado estão em
[Ecosystem Data Directory Hierarchy Standard](./ecosystem-data-directory-hierarchy-standard.md).

## Exemplos reais

- `essential-repository` — `Commons.Module`, `Runtime.Module`, `Main.Module`
  (publica `repo`, `supervisor`, `mytoolkit`).
- `ecosystem-core-repository` — publica `executor-manager`, `executor`,
  `explorer`, `eco-panel`, `executor-panel`, `mypkg`, `run`.
- `applications-repository` — publica `api-designer-webapp` (APP),
  `api-designer-desktop` (DESKTOP), `developer`, `sources`.
