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

## Fontes (`sources.json`) e instalação

Um repositório é descoberto/instalado a partir de uma **fonte** registrada no
`sources.json` do ecossistema:

| `sourceType` | Origem | Parâmetros |
|--------------|--------|-----------|
| `LOCAL_FS` | Sistema de arquivos local | `localPath` |
| `GITHUB_RELEASE` | Release do GitHub | `repoOwner`, `repoName` |
| `GOOGLE_DRIVE` | `.tar.gz` no Google Drive | `fileId` |

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
