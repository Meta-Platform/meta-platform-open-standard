# Ecosystem Installation Profile Standard

Define o formato de um **Installation Profile** (`*.install.json`) — o arquivo que
descreve **o que** instalar (repositórios e executáveis) e **onde**, consumido
pelo [Setup Wizard](https://github.com/Meta-Platform/meta-platform-setup-wizard-command-line)
ao montar um [EcosystemData](./ecosystem-data-directory-hierarchy-standard.md).

## Formato

```json
{
    "installationDataDir": "~/EcosystemData",
    "repositoriesToInstall": [
        {
            "namespace": "EssentialRepo",
            "sourceType": "LOCAL_FS",
            "executablesToInstall": ["supervisor", "mytoolkit", "repo"]
        }
    ]
}
```

| Campo | Descrição |
|-------|-----------|
| `installationDataDir` | Diretório `EcosystemData` de destino (aceita `~`). Pode ser sobrescrito por `--installation-path`. |
| `repositoriesToInstall[]` | Repositórios a instalar. |
| `repositoriesToInstall[].namespace` | Nome do [Repository](../concepts/repository.md); deve existir nas fontes (`repository-sources.json` / `sources.json`). |
| `repositoriesToInstall[].sourceType` | Origem: `LOCAL_FS`, `GITHUB_RELEASE` ou `GOOGLE_DRIVE`. |
| `repositoriesToInstall[].executablesToInstall` | [Executáveis](../concepts/executable.md) (de `applications.json` do repositório) a instalar em `EcosystemData/executables/`. |

## Convenção de nomes (perfis de referência)

Os perfis distribuídos com o wizard seguem `[dev-]<origem>-<escopo>`:

- **Destino:** `dev-*` → `~/Workspaces/.../EcosystemData` (desenvolvimento); os
  demais → `~/EcosystemData`.
- **Origem:** `*localfs*` → `LOCAL_FS`; `*github-release*` → `GITHUB_RELEASE`.
- **Escopo:** `minimal` = só `EssentialRepo`; `standard` = `EssentialRepo` +
  `EcosystemCoreRepo`; `full` = + `PlatformApplicationsRepo`.

> O **conjunto de perfis efetivamente registrados** e o comportamento do CLI são
> da implementação de referência (e podem divergir dos arquivos presentes no
> diretório). Sempre confira no repositório do
> [Setup Wizard](https://github.com/Meta-Platform/meta-platform-setup-wizard-command-line)
> (README + `docs/known-issues.md`).

## Relação com o ecossistema

O Setup Wizard lê o profile, resolve cada `namespace` para uma fonte, baixa/copia
o repositório para `EcosystemData/repos/`, registra-o em `repositories.json` e
cria os scripts em `EcosystemData/executables/` para os `executablesToInstall`.
Ver [Ecosystem Data Directory Hierarchy Standard](./ecosystem-data-directory-hierarchy-standard.md).
