# Execution Params File Standard

## Visão Geral

É um padrão de arquivo JSON que descreve um processo completo a ser executado, definindo a sequência de objetos (loaders) que devem ser processados, suas dependências e configurações.

## Estrutura Raiz

O arquivo deve ser um array JSON contendo objetos que representam diferentes tipos de carregadores (`objectLoaderType`).

```json
[
  {
    "objectLoaderType": "tipo-do-carregador",
    "staticParameters": { ... },
    "activationRules": { ... },
    "linkedParameters": { ... },
    "children": [ ... ]
  }
]
```


---

## Campos Comuns

### `staticParameters`
Objeto contendo parâmetros estáticos que não mudam durante a execução. Variam conforme o tipo de loader.

### `activationRules` (opcional)
Define as condições necessárias para que um objeto seja ativado. Utiliza operadores lógicos:
- `&&`: AND lógico (todas as condições devem ser verdadeiras)
- `||`: OR lógico (ao menos uma condição deve ser verdadeira)

**Estrutura:**
```json
"activationRules": {
  "&&": [
    {
      "property": "caminho.da.propriedade",
      "=": "valor_esperado"
    }
  ]
}
```

### `linkedParameters` (opcional)
Mapeia referências para outros componentes que serão injetados como parâmetros:
```json
"linkedParameters": {
  "nodejsPackageHandler": "@/service-name",
  "customParam": "@@/local-reference"
}
```

### `agentLinkRules` (opcional)
Define requisitos de dependência entre componentes:
```json
"agentLinkRules": [
  {
    "referenceName": "@/referenced-component",
    "requirement": {
      "&&": [
        {
          "property": "params.tag",
          "=": "@/referenced-component"
        },
        {
          "property": "status",
          "=": "ACTIVE"
        }
      ]
    }
  }
]
```

### `children` (opcional)
Array de objetos filhos (apenas para `application-instance`). Define a hierarquia de componentes dentro de uma aplicação.

---

## Convenções de Nomenclatura

- **Namespace Global**: `@/identificador` - Referência a componentes globais (pacotes, aplicações)
- **Referência Local**: `@@/identificador` - Referência a componentes filhos dentro do mesmo contexto
- **Tags de Status**: Valores comuns: `FINISHED`, `ACTIVE`, `PENDING`, `FAILED`
- **Diretórios**: Usar caminhos completos ou relativos (~/ para home)

---

## Fluxo de Execução

1. **Instalação de Dependências**: Executar todos os loaders `install-nodejs-package-dependencies`
2. **Carregamento de Pacotes**: Ativar loaders `nodejs-package` conforme suas `activationRules`
3. **Instanciação de Aplicação**: Criar instância `application-instance` com suas dependências
4. **Instanciação de Serviços**: Criar `service-instance` filhas conforme regras de vinculação
5. **Criação de Endpoints**: Ativar `endpoint-instance` que expõem a API da aplicação

---

## Boas Práticas

- Sempre validar a sintaxe JSON antes de usar o arquivo
- Garantir que todas as referências (`@/` e `@@/`) existem
- Manter a hierarquia consistente em `children`
- Usar caminhos absolutos para máximo portabilidade
- Documentar parâmetros customizados em comentários (fora do JSON)
- Verificar `activationRules` para evitar deadlocks de dependência