# Dependency Resolution Standard

Define como as dependências entre [packages](../concepts/package.md) são
declaradas e resolvidas no Meta Platform.

## Princípio: dependência por namespace

Packages dependem uns dos outros **sempre por namespace**, **nunca** por caminho
relativo no sistema de arquivos. Isso desacopla os packages do disco e permite
que repositórios sejam instalados em qualquer lugar do `EcosystemData`.

## Prefixos de namespace

| Prefixo | Significado | Exemplo |
|---------|-------------|---------|
| `@/` | Package por namespace no conjunto de repositórios instalados (declarado dentro de um repositório, resolvido globalmente no `EcosystemData`) | `@/json-file-utilities.lib` |
| `@@/` | Instância/serviço dentro do mesmo contexto de execução | `@@/server-service` |
| `@//` | Referência interna de boot do próprio package | `@//command-group`, `@//endpoint-group` |

## Onde as dependências são declaradas

- **`boot.json`** → `bound-params` (dependências de um executável) e `services`
  (ver [Executable Manifest Standard](./executable-manifest-standard.md)).
- **`command-group.json`** → `parametersToLoad` (quais `bound-params` cada comando
  recebe).
- **`services.json`** → `bound-params` de cada serviço (`?nome` = opcional).

## Como a resolução acontece

1. O [Package Executor](./package-executor-rpc-standard.md) lista todos os
   packages dos repositórios instalados (varrendo a hierarquia
   `Module/Layer/[Group/]Package`).
2. Constrói o **grafo de metadados** (`metadata-hierarchy.json`) a partir do
   package raiz, resolvendo cada namespace para o package correspondente
   (dependências diretas e transitivas).
3. O grafo é traduzido em [`execution-params.json`](./packages/execution-params-standard.md),
   onde as ligações entre unidades aparecem como `linkedParameters` e
   `agentLinkRules`.
4. No [Runtime Environment](../concepts/runtime-environment.md), o **Task
   Executor** instancia cada unidade resolvendo essas ligações para as instâncias
   já ativas.

## Em runtime: `linkedParameters` e `agentLinkRules`

No `execution-params.json`, uma unidade declara o que precisa de outra via:

- `linkedParameters` — injeta a referência de outra unidade como parâmetro;
- `agentLinkRules` — condições (sobre `params`/`status`) que a unidade
  referenciada deve satisfazer para a ligação ser válida.

Ver [Execution Params Standard](./packages/execution-params-standard.md).

> **Limitação conhecida (estado atual):** o mecanismo de resolução/`SmartRequire`
> e as dependências internas entre repos em múltiplos níveis ainda são um item de
> melhoria do projeto (planejamento interno).
