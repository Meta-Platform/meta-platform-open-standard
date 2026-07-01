# Package (Pacote)

> Definição formal e independente de implementação. Para um tutorial prático de
> criação, veja o [Guia: Criar um Pacote](https://github.com/Meta-Platform/.github/blob/main/docs/GUIA-CRIAR-PACOTE.md).

## Definição

Um **package** (pacote) é a **unidade atômica** do Meta Platform: a menor unidade
autodescritiva que pode ser reutilizada ou executada. Tudo que é executável ou
reutilizável na plataforma é empacotado em um package.

Um package é uma pasta cujo **nome termina com um sufixo de tipo** e que contém,
obrigatoriamente, uma pasta [`metadata/`](./metadata.md) que o torna
compreensível para a plataforma **sem necessidade de executá-lo**.

## Tipos de package (sufixos)

O sufixo do nome da pasta determina o tipo do package. O conjunto canônico de
tipos é definido em
[`ecosystem-defaults.json`](../specifications/metadados/ecosystem-defaults.json)
na chave `REPOS_CONF_EXTLIST_PKG_TYPE`
(`app|cli|webapp|desktopapp|webgui|webservice|service|lib`), acrescido do tipo
`nativelib` para bibliotecas nativas:

| Sufixo | Tipo | Descrição |
|--------|------|-----------|
| `.lib` | Library | Biblioteca reutilizável (JavaScript). |
| `.nativelib` | Native library | Biblioteca nativa (addon C++/Rust via `binding.gyp`/`Cargo.toml`). |
| `.cli` | Command-line | Aplicação de linha de comando; expõe um executável. |
| `.service` | Service | Serviço de back-end de longa duração. |
| `.webservice` | Web service | Serviço HTTP / API. |
| `.webgui` | Web GUI | Interface web (front-end). |
| `.webapp` | Web application | Aplicação web (webgui + webservice integrados). |
| `.desktopapp` | Desktop application | Aplicação desktop nativa; abre uma ou mais janelas [Electron](https://www.electronjs.org/). Tipicamente encapsula uma app web local que sobe junto (`loadURL`); também suporta HTML local (`loadFile`). |
| `.app` | Application | Aplicação/instância do ecossistema. |

## Identidade e dependências por namespace

Todo package declara um **namespace** em `metadata/package.json`. Os prefixos têm
significado fixo:

- `@/` — referência a um package por **namespace** no conjunto de repositórios
  instalados. A declaração normalmente aparece dentro de um repositório, mas a
  resolução acontece globalmente no `EcosystemData` (ex.:
  `@/repository-manager.cli`).
- `@@/` — referência a uma **instância/serviço** dentro do mesmo contexto de
  execução (ex.: `@@/server-service`).
- `@//` — referência **interna de boot** do próprio package (ex.:
  `@//command-group`).

> **Regra fundamental:** dependências entre packages são feitas **sempre por
> namespace**, nunca por caminho relativo ao sistema de arquivos. A resolução
> acontece em tempo de execução, dentro do ecossistema. Isso é o que permite que
> packages se liguem sem acoplamento físico ao disco.

## Anatomia

```
<nome>.<sufixo>/
├── metadata/          # obrigatório — torna o package compreensível à plataforma
│   ├── package.json   # namespace (obrigatório em todo package)
│   ├── boot.json      # (executáveis) o que o package expõe + bound-params
│   ├── command-group.json   # (CLI) árvore de comandos
│   └── startup-params.json   # parâmetros de inicialização
├── src/               # código-fonte (módulos em PascalCase)
├── package.json       # identidade e dependências NPM do package
└── README.md          # descrição do package (convenção dos packages oficiais)
```

Uma biblioteca (`.lib`) no mínimo precisa apenas de `metadata/package.json`
(namespace) e do código em `src/`. Pacotes executáveis (`.cli`, `.app`, …)
acrescentam `boot.json` e os demais metadados conforme o tipo. A definição
formal de cada arquivo está em
[Package Metadata Standard](../specifications/package-metadata-standard.md)
(visão de conceito em [Metadata](./metadata.md)).

Um package que expõe um ponto de entrada gera um [Executable](./executable.md).

## Hierarquia

Packages não ficam soltos: são organizados na hierarquia
`Repository → Module (*.Module) → Layer (*.layer) → (Group (*.group) →) Package`.
Veja [Repository](./repository.md) e
[Module / Layer / Group](./module-layer-group.md).

## Ciclo de vida (visão geral)

1. Um package é distribuído dentro de um [Repository](./repository.md).
2. O [Package Executor](../specifications/package-executor-rpc-standard.md) cria um
   [Runtime Environment](./runtime-environment.md) isolado, resolve as
   dependências por namespace (ver
   [Dependency Resolution Standard](../specifications/dependency-resolution-standard.md))
   e gera o [`execution-params.json`](../specifications/packages/execution-params-standard.md).
3. O **Task Executor** instancia cada unidade através do
   [object loader](./tipos-de-object-loader.md) correspondente.
