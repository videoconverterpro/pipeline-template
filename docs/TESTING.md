# 🧪 Testing - Guia de Testes

Documentação completa sobre a action de testes genérica para projetos Node.js.

## 📋 Visão Geral

A action `v1/nodejs/24/test` é **framework-agnostic** e suporta múltiplos tipos de testes através dos scripts definidos no `package.json` do seu projeto.

## 🎯 Tipos de Teste Suportados

### 1. Testes Unitários (`unit`)
- **Script**: `pnpm test`
- **Propósito**: Testar funções, classes e métodos isoladamente
- **Velocidade**: ⚡ Rápido (milissegundos)
- **Exemplos**: Jest, Vitest, Mocha

### 2. Testes de Integração (`integration`)
- **Script**: `pnpm test:integration`
- **Propósito**: Testar comunicação entre módulos/serviços
- **Velocidade**: 🟡 Moderado (segundos)
- **Exemplos**: Testes de banco de dados, APIs externas

### 3. Testes E2E (`e2e`)
- **Script**: `pnpm test:e2e`
- **Propósito**: Testar fluxo completo da aplicação
- **Velocidade**: 🐢 Lento (minutos)
- **Exemplos**: Supertest (NestJS), Playwright (Next.js), Cypress

### 4. Cobertura de Código (`coverage`)
- **Script**: `pnpm test:cov`
- **Propósito**: Gerar relatório de cobertura de testes
- **Output**: `coverage/` (automaticamente enviado como artifact)
- **Retenção**: 7 dias

## 🔧 Inputs Disponíveis

| Input | Tipo | Padrão | Descrição |
|-------|------|--------|-----------|
| `unit` | boolean | `true` | Executar testes unitários |
| `integration` | boolean | `false` | Executar testes de integração |
| `e2e` | boolean | `false` | Executar testes e2e |
| `coverage` | boolean | `false` | Gerar relatório de cobertura + upload artifact |

## 📖 Exemplos de Uso

### Exemplo 1: Apenas Testes Unitários (padrão)

```yaml
- name: Test - Unitários
  uses: videoconverterpro/pipeline-template/v1/nodejs/24/test@main
```

### Exemplo 2: Unitários + E2E (recomendado para CI)

```yaml
- name: Test - Unitários e E2E
  uses: videoconverterpro/pipeline-template/v1/nodejs/24/test@main
  with:
    unit: 'true'
    e2e: 'true'
```

### Exemplo 3: Todos os Tipos de Teste

```yaml
- name: Test - Completo
  uses: videoconverterpro/pipeline-template/v1/nodejs/24/test@main
  with:
    unit: 'true'
    integration: 'true'
    e2e: 'true'
    coverage: 'false'
```

### Exemplo 4: Apenas E2E (para pipelines específicos)

```yaml
- name: Test - E2E Smoke Tests
  uses: videoconverterpro/pipeline-template/v1/nodejs/24/test@main
  with:
    unit: 'false'
    e2e: 'true'
```

### Exemplo 5: Com Cobertura de Código

```yaml
- name: Test - Com Coverage
  uses: videoconverterpro/pipeline-template/v1/nodejs/24/test@main
  with:
    unit: 'true'
    integration: 'true'
    e2e: 'true'
    coverage: 'true'
```

## 🏗️ Configuração do Projeto

### Requisitos no `package.json`

```json
{
  "scripts": {
    "test": "jest",                                    // Obrigatório para unit=true
    "test:integration": "jest --config jest.int.json", // Obrigatório para integration=true
    "test:e2e": "jest --config jest-e2e.json",         // Obrigatório para e2e=true
    "test:cov": "jest --coverage"                      // Obrigatório para coverage=true
  }
}
```

**Nota:** Se um script não existir, a action exibe aviso e continua (não falha).

## 🎭 Frameworks Compatíveis

### NestJS
```json
{
  "scripts": {
    "test": "jest",
    "test:e2e": "jest --config ./test/jest-e2e.json",
    "test:cov": "jest --coverage"
  }
}
```

### Next.js
```json
{
  "scripts": {
    "test": "vitest",
    "test:e2e": "playwright test",
    "test:cov": "vitest --coverage"
  }
}
```

### Express/Fastify
```json
{
  "scripts": {
    "test": "mocha",
    "test:integration": "mocha test/integration",
    "test:e2e": "supertest test/e2e"
  }
}
```

## 📊 Performance

| Tipo | Duração Média | Cache? | Quando Usar |
|------|---------------|--------|-------------|
| Unit | 1-5s | ❌ | Todo push |
| Integration | 5-15s | ❌ | Todo push |
| E2E | 15-60s | ❌ | Todo push/PR |
| Coverage | +5-10s | ✅ (artifact) | PRs ou main |

## 🚀 Estratégias Recomendadas

### Para Branches de Feature
```yaml
- uses: videoconverterpro/pipeline-template/v1/nodejs/24/test@main
  with:
    unit: 'true'
    integration: 'false'
    e2e: 'false'
    coverage: 'false'
```
**Motivo**: Feedback rápido durante desenvolvimento

### Para Pull Requests
```yaml
- uses: videoconverterpro/pipeline-template/v1/nodejs/24/test@main
  with:
    unit: 'true'
    integration: 'true'
    e2e: 'true'
    coverage: 'true'
```
**Motivo**: Validação completa antes do merge

### Para Branch Main/Production
```yaml
- uses: videoconverterpro/pipeline-template/v1/nodejs/24/test@main
  with:
    unit: 'true'
    integration: 'true'
    e2e: 'true'
    coverage: 'false'
```
**Motivo**: Garantir qualidade sem overhead de coverage

## 🔍 Troubleshooting

### Erro: "Script 'test:e2e' não encontrado"
**Causa**: O script não existe no `package.json`  
**Solução**: Adicione o script ou ajuste o input para `e2e: 'false'`

### Testes falhando no CI mas passando localmente
**Possíveis causas:**
- Dependências de banco de dados não disponíveis
- Variáveis de ambiente faltando
- Timeouts muito curtos
- Cache de `node_modules` corrompido

**Soluções:**
```yaml
# Adicionar serviços necessários
services:
  postgres:
    image: postgres:16
    env:
      POSTGRES_PASSWORD: test

# Configurar variáveis de ambiente
env:
  DATABASE_URL: postgresql://postgres:test@localhost:5432/test
  NODE_ENV: test
```

### Coverage report não sendo gerado
**Causa**: Configuração de coverage faltando  
**Solução**: Configure no `jest.config.js`:
```js
module.exports = {
  coverageDirectory: 'coverage',
  collectCoverageFrom: [
    'src/**/*.{js,ts}',
    '!src/**/*.spec.{js,ts}'
  ]
}
```

## 📦 Artifacts

Quando `coverage: 'true'`, um artifact é criado:
- **Nome**: `coverage-<sha>`
- **Conteúdo**: Pasta `coverage/` completa
- **Retenção**: 7 dias
- **Download**: GitHub Actions → Run → Artifacts

## 🔗 Integração com Ferramentas Externas

### Codecov
```yaml
- uses: videoconverterpro/pipeline-template/v1/nodejs/24/test@main
  with:
    coverage: 'true'

- uses: codecov/codecov-action@v4
  with:
    files: ./coverage/coverage-final.json
```

### SonarQube
```yaml
- uses: videoconverterpro/pipeline-template/v1/nodejs/24/test@main
  with:
    coverage: 'true'

- uses: sonarsource/sonarcloud-github-action@master
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

## 📚 Recursos Adicionais

- [Jest Documentation](https://jestjs.io/)
- [Vitest Documentation](https://vitest.dev/)
- [Playwright Documentation](https://playwright.dev/)
- [Supertest Documentation](https://github.com/ladjs/supertest)

## 📝 Licença

**Proprietário** - Bruno Roberto Morillo  
CPF: 460.876.598-11  
© 2025 VideoConverterPro - Todos os direitos reservados
