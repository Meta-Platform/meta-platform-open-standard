# Executable

> Conceito do Meta Platform Open Standard. Veja também
> [Package](./package.md), [Runtime Environment](./runtime-environment.md) e a
> especificação [Executable Manifest Standard](../specifications/executable-manifest-standard.md).

## Definição

Um **Executable** (executável) é um **ponto de entrada nomeado** que um
[package](./package.md) expõe ao ecossistema. É o que o usuário invoca no
terminal (uma `CLI`) ou o que o ecossistema sobe como aplicação/serviço (`APP`,
`Service`, `WebApp`).

Um executável **não** é o mesmo que o package: um package descreve *o que existe*;
o executável descreve *o que pode ser iniciado* a partir dele.

## Onde os executáveis são declarados

Em dois níveis complementares:

1. **No package** — o arquivo `metadata/boot.json` declara o que o package
   expõe ao ser carregado. Para uma CLI, lista `executables` com `executableName`
   (o nome do comando), a `dependency` que o constrói (ex.: `@//command-group`) e
   os `bound-params` (dependências resolvidas por namespace). Ver
   [Executable Manifest Standard](../specifications/executable-manifest-standard.md).

2. **No repositório** — o arquivo `metadata/applications.json` lista os
   executáveis que o repositório **publica** para o ecossistema, associando:
   - `executable` — o nome instalado em `EcosystemData/executables/` (que precisa
     estar no `PATH`);
   - `packageNamespace` — o caminho do package dentro do repositório
     (`Module/Layer/[Group/]Package`);
   - `supervisorSocketFileName` — o nome do [supervisor socket](./runtime-environment.md)
     criado quando o executável roda;
   - `appType` — o tipo (`CLI`, `APP`, …).
   Ver [Repository Metadata Standard](../specifications/repository-metadata-standard.md).

## Tipos de executável e tipos de package

O **tipo do executável** (`appType` em `applications.json`) está ligado ao
**sufixo do package** que o origina:

| `appType` | Sufixo típico do package | O que é |
|-----------|--------------------------|---------|
| `CLI` | `.cli` | Comando de linha de comando. |
| `APP` | `.app`, `.webapp` | Aplicação/instância do ecossistema (pode subir serviços e endpoints web). |

> Os sufixos de package (`.lib`, `.cli`, `.service`, `.webservice`, `.webgui`,
> `.webapp`, `.app`, `.nativelib`) estão descritos em [Package](./package.md).
> Nem todo package gera um executável: bibliotecas (`.lib`) são consumidas por
> outros packages e não aparecem em `applications.json`.

## Como um executável é instalado e iniciado

1. O [Setup Wizard](https://github.com/Meta-Platform/meta-platform-setup-wizard-command-line)
   (ou a CLI `repo`) lê `applications.json` e cria um script em
   `EcosystemData/executables/` para cada `executable` do
   `executablesToInstall` do perfil.
2. Ao ser invocado, o script chama o
   [Package Executor](../specifications/package-executor-rpc-standard.md), que cria
   um [Runtime Environment](./runtime-environment.md) e executa o package.
3. Se um `supervisorSocketFileName` estiver associado, o processo expõe um
   [supervisor socket](../specifications/supervisor-socket-standard.md) (gRPC sobre
   Unix socket) para supervisão.
