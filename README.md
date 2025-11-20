# 🚀 Pipeline Template - Workflows Reutilizáveis

[![License](https://img.shields.io/badge/License-Proprietary-red.svg)](LICENSE)
[![Node.js](https://img.shields.io/badge/Node.js-24+-green.svg)](https://nodejs.org/)
[![pnpm](https://img.shields.io/badge/pnpm-10+-blue.svg)](https://pnpm.io/)

Workflows reutilizáveis do GitHub Actions para projetos Node.js/NestJS/TypeScript.

**100% genéricos** - Use em qualquer projeto sem duplicar código!

---

## 📦 Workflows Disponíveis

### 1. 🔧 Setup Node.js + pnpm + Cache
**Arquivo:** `.github/workflows/setup-node-pnpm.yml`

Configura ambiente Node.js com pnpm e sistema de cache multi-camadas.

**Características:**
- ✅ Node.js e pnpm instalados
- ✅ Cache inteligente (pnpm store, node_modules, Prisma engines)
- ✅ 85% mais rápido com cache hit
- ✅ Detecta e regenera Prisma Client automaticamente

**Uso:**
```yaml
jobs:
  setup:
    uses: videoconverterpro/pipeline-template/.github/workflows/setup-node-pnpm.yml@main
    with:
      node-version: '24'      # Opcional, padrão: '24'
      pnpm-version: '10'      # Opcional, padrão: '10'
      working-directory: '.'  # Opcional, padrão: '.'
```

---

### 2. ✅ Quality Check (Prettier + ESLint)
**Arquivo:** `.github/workflows/quality-check.yml`

Valida formatação e linting do código.

**Características:**
- ✅ Prettier format check
- ✅ ESLint com --max-warnings 0 (strict)
- ✅ Fail-fast (falha imediatamente se código não conforme)
- ✅ Modo check-only (não modifica código)

**Uso:**
```yaml
jobs:
  quality:
    needs: setup  # Executar após setup
    uses: videoconverterpro/pipeline-template/.github/workflows/quality-check.yml@main
    with:
      working-directory: '.'          # Opcional, padrão: '.'
      format-script: 'format:check'   # Opcional, padrão: 'format:check'
      lint-script: 'lint:check'       # Opcional, padrão: 'lint:check'
```

**Pré-requisitos no `package.json`:**
```json
{
  "scripts": {
    "format:check": "prettier --check .",
    "lint:check": "eslint . --max-warnings 0"
  }
}
```

---

### 3. 🏗️ Build + Artifact Upload
**Arquivo:** `.github/workflows/build-app.yml`

Compila aplicação e faz upload do artifact.

**Características:**
- ✅ Executa script de build configurado
- ✅ Upload automático de artifact (dist/)
- ✅ Retenção configurável (padrão: 7 dias)
- ✅ Suporta NestJS, React, Next.js, Vite, etc.

**Uso:**
```yaml
jobs:
  build:
    needs: [setup, quality]
    uses: videoconverterpro/pipeline-template/.github/workflows/build-app.yml@main
    with:
      working-directory: '.'       # Opcional, padrão: '.'
      build-script: 'build'        # Opcional, padrão: 'build'
      dist-folder: 'dist'          # Opcional, padrão: 'dist'
      artifact-retention: 7        # Opcional, dias de retenção
```

**Pré-requisitos no `package.json`:**
```json
{
  "scripts": {
    "build": "nest build"  // ou "tsc", "vite build", etc.
  }
}
```

---

### 4. 🚀 Pipeline Completa de Validação
**Arquivo:** `.github/workflows/validate-pipeline.yml`

Pipeline completa orquestrando setup → quality → build.

**Características:**
- ✅ Execução sequencial: Setup → Quality → Build
- ✅ Todas as validações em uma única chamada
- ✅ Altamente customizável
- ✅ Perfeito para branches de homologação

**Uso:**
```yaml
jobs:
  validate:
    uses: videoconverterpro/pipeline-template/.github/workflows/validate-pipeline.yml@main
    with:
      node-version: '24'
      pnpm-version: '10'
      working-directory: '.'
      format-script: 'format:check'
      lint-script: 'lint:check'
      build-script: 'build'
      dist-folder: 'dist'
      artifact-retention: 7
```

---

## 🎯 Exemplos Práticos

### Exemplo 1: Validação de Homologação

Crie `.github/workflows/homolog.yml` no seu projeto:

```yaml
name: "🔍 Validação Homologação"

on:
  push:
    branches: [homolog]
  pull_request:
    branches: [homolog]

jobs:
  validate:
    name: "🔍 Validar Código"
    uses: videoconverterpro/pipeline-template/.github/workflows/validate-pipeline.yml@main
    with:
      node-version: '24'
      pnpm-version: '10'
```

**Resultado:**
- ✅ Código formatado (Prettier)
- ✅ Código lintado (ESLint)
- ✅ Build funcionando
- ✅ Artifact gerado

---

### Exemplo 2: Deploy de Produção

Crie `.github/workflows/production.yml` no seu projeto:

```yaml
name: "🚀 Deploy Produção"

on:
  push:
    branches: [main]

jobs:
  # Validação completa
  validate:
    name: "🔍 Validar"
    uses: videoconverterpro/pipeline-template/.github/workflows/validate-pipeline.yml@main
    with:
      node-version: '24'
      pnpm-version: '10'
      artifact-retention: 30  # Produção: retenção maior
  
  # Deploy (seu job customizado)
  deploy:
    name: "🚀 Deploy VPS"
    needs: validate
    runs-on: ubuntu-latest
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: dist-${{ github.sha }}
      
      - name: Deploy para VPS
        run: |
          # Seu script de deploy aqui
          scp -r dist/ user@vps:/app
          ssh user@vps "pm2 restart app"
```

---

### Exemplo 3: Monorepo (Múltiplos Projetos)

```yaml
name: "🔍 Validação Monorepo"

on: [push, pull_request]

jobs:
  # API Backend
  validate-api:
    name: "🔍 API"
    uses: videoconverterpro/pipeline-template/.github/workflows/validate-pipeline.yml@main
    with:
      working-directory: 'packages/api'
      build-script: 'build'
  
  # Frontend App
  validate-app:
    name: "🔍 App"
    uses: videoconverterpro/pipeline-template/.github/workflows/validate-pipeline.yml@main
    with:
      working-directory: 'packages/app'
      build-script: 'build'
      dist-folder: 'out'  # Next.js usa 'out'
```

---

## 🔧 Configuração do Projeto

### Pré-requisitos

Seu projeto deve ter os seguintes scripts no `package.json`:

```json
{
  "scripts": {
    "format:check": "prettier --check .",
    "lint:check": "eslint . --max-warnings 0",
    "build": "nest build"  // ou tsc, vite build, etc.
  }
}
```

### Estrutura Esperada

```
seu-projeto/
├── .github/
│   └── workflows/
│       ├── homolog.yml      # Usa pipeline-template
│       └── production.yml   # Usa pipeline-template
├── package.json             # Com scripts acima
├── pnpm-lock.yaml          # Lock file do pnpm
└── src/                     # Seu código-fonte
```

---

## 📊 Performance

### Cache Multi-Camadas

| Layer | O que Cacheia | Economia | Invalidação |
|-------|---------------|----------|-------------|
| 1 | pnpm store | ~60s | pnpm-lock.yaml muda |
| 2 | node_modules | ~60s | pnpm-lock.yaml muda |
| 3 | Prisma engines | ~30s | schema.prisma muda |

**Total: ~85% mais rápido com cache hit!**

### Tempos Médios

| Cenário | Tempo | Cache |
|---------|-------|-------|
| **Primeira execução** | ~165s | ❌ MISS |
| **Cache hit total** | ~25s | ✅ HIT |
| **Apenas lock mudou** | ~80s | ⚠️ Fallback |

---

## 🎛️ Customizações Avançadas

### Scripts Customizados

Se seu projeto usa nomes diferentes:

```yaml
jobs:
  validate:
    uses: videoconverterpro/pipeline-template/.github/workflows/validate-pipeline.yml@main
    with:
      format-script: 'prettier:validate'  # Ao invés de format:check
      lint-script: 'eslint:strict'        # Ao invés de lint:check
      build-script: 'compile'             # Ao invés de build
```

### Retenção de Artifacts

```yaml
with:
  artifact-retention: 1   # Dev: 1 dia
  artifact-retention: 7   # Homolog: 7 dias (padrão)
  artifact-retention: 30  # Produção: 30 dias
  artifact-retention: 90  # Compliance: 90 dias
```

### Versões do Node.js/pnpm

```yaml
with:
  node-version: '20'  # LTS anterior
  node-version: '24'  # LTS atual (padrão)
  node-version: '25'  # Latest
  
  pnpm-version: '9'   # Versão anterior
  pnpm-version: '10'  # Versão atual (padrão)
```

---

## 🐛 Troubleshooting

### Erro: "pnpm não encontrado"

**Causa:** Tentando usar cache nativo do setup-node antes da instalação do pnpm.

**Solução:** Os workflows já corrigem isso! Não use `cache: 'pnpm'` no setup-node.

### Erro: "Script not found: format:check"

**Causa:** Scripts não configurados no package.json.

**Solução:** Adicione os scripts ou customize os nomes:
```yaml
with:
  format-script: 'seu-script-format'
```

### Cache não está funcionando

**Causa:** pnpm-lock.yaml não commitado ou .gitignore bloqueando.

**Solução:** Commit o lock file:
```bash
git add pnpm-lock.yaml
git commit -m "chore: adicionar lock file"
```

### Build falha no CI mas funciona localmente

**Causa:** Diferenças de ambiente ou dependências faltando.

**Solução:** 
1. Verifique Node.js/pnpm versions
2. Execute localmente: `pnpm install --frozen-lockfile`
3. Cheque logs do CI para erro específico

---

## 📚 Documentação Adicional

- [GitHub Actions - Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [pnpm Documentation](https://pnpm.io/)
- [Actions Cache](https://github.com/actions/cache)

---

## 📄 Licença

**Propriedade Privada**  
© 2025 VideoConverterPro - Todos os direitos reservados  
**Proprietário:** Bruno Roberto Morillo  
**CPF:** 460.876.598-11

Este código é **totalmente privado** e **não possui licença open-source**.  
Uso restrito aos projetos da organização VideoConverterPro.

---

## 🤝 Contribuição

Contribuições são restritas aos membros da organização VideoConverterPro.

Para sugestões ou melhorias, abra uma issue interna.

---

## 🎯 Roadmap

- [ ] Workflow para deploy automático (VPS, Docker, Kubernetes)
- [ ] Workflow para testes (unit, e2e, coverage)
- [ ] Workflow para release automation
- [ ] Suporte para outros package managers (npm, yarn)
- [ ] Notificações (Slack, Discord, Email)

---

**Feito com ❤️ por VideoConverterPro**
