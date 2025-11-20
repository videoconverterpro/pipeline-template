# 🚀 Pipeline Template - Composite Actions Reutilizáveis

[![License](https://img.shields.io/badge/License-Proprietary-red.svg)](LICENSE)
[![Node.js](https://img.shields.io/badge/Node.js-24+-green.svg)](https://nodejs.org/)
[![pnpm](https://img.shields.io/badge/pnpm-10+-blue.svg)](https://pnpm.io/)

Composite Actions reutilizáveis do GitHub Actions para projetos Node.js/NestJS/TypeScript.

**100% genéricos** - Use em qualquer projeto sem duplicar código!  
**✅ Funciona com repositórios privados** - Sem necessidade de GitHub Enterprise!

---

## 📦 Actions Disponíveis

### 1. 🔧 Setup Node.js + pnpm + Cache

**Arquivo:** `.github/actions/setup-node-pnpm/action.yml`

Configura ambiente Node.js com pnpm e sistema de cache multi-camadas.

**Uso:**

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: videoconverterpro/pipeline-template/.github/actions/setup-node-pnpm@main
    with:
      node-version: '24'      # Opcional, padrão: '24'
      pnpm-version: '10'      # Opcional, padrão: '10'
```

---

### 2. ✅ Quality Check (Prettier + ESLint)

**Arquivo:** `.github/actions/quality-check/action.yml`

Valida formatação e linting do código.

**Uso:**

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: videoconverterpro/pipeline-template/.github/actions/setup-node-pnpm@main
  - uses: videoconverterpro/pipeline-template/.github/actions/quality-check@main
    with:
      format-script: 'format:check'   # Opcional
      lint-script: 'lint:check'       # Opcional
```

---

### 3. 🏗️ Build + Artifact Upload

**Arquivo:** `.github/actions/build-app/action.yml`

Compila aplicação e faz upload do artifact.

**Uso:**

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: videoconverterpro/pipeline-template/.github/actions/setup-node-pnpm@main
  - uses: videoconverterpro/pipeline-template/.github/actions/build-app@main
    with:
      build-script: 'build'        # Opcional
      dist-folder: 'dist'          # Opcional
      artifact-retention: 7        # Opcional
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

jobs:
  validate:
    name: "🔍 Validar Código"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: videoconverterpro/pipeline-template/.github/actions/setup-node-pnpm@main
        with:
          node-version: '24'
          pnpm-version: '10'
      
      - uses: videoconverterpro/pipeline-template/.github/actions/quality-check@main
      
      - uses: videoconverterpro/pipeline-template/.github/actions/build-app@main
        with:
          artifact-retention: 7
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
  validate:
    name: "🔍 Validar"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: videoconverterpro/pipeline-template/.github/actions/setup-node-pnpm@main
      - uses: videoconverterpro/pipeline-template/.github/actions/quality-check@main
      - uses: videoconverterpro/pipeline-template/.github/actions/build-app@main
        with:
          artifact-retention: 30  # Produção: retenção maior
  
  deploy:
    name: "🚀 Deploy VPS"
    needs: validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist-${{ github.sha }}
      
      - name: Deploy
        run: |
          scp -r dist/ user@vps:/app
          ssh user@vps "pm2 restart app"
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
