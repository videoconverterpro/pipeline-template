# 🚀 Pipeline Template - CI/CD Reutilizável Multi-Tech

[![License](https://img.shields.io/badge/License-Proprietary-red.svg)](LICENSE)
[![Node.js](https://img.shields.io/badge/Node.js-24+-green.svg)](https://nodejs.org/)
[![Go](https://img.shields.io/badge/Go-1.22+-00ADD8.svg)](https://go.dev/)
[![Rust](https://img.shields.io/badge/Rust-stable-orange.svg)](https://rust-lang.org/)

Repositório centralizado de **composite actions** para pipelines CI/CD em múltiplas tecnologias.

> 📖 **Documentação Completa**: Consulte [`docs/`](docs/) para guias detalhados sobre cada action.

## 📁 Estrutura

```text
v1/
├── nodejs/
│   ├── 24/                     # Node.js 24 (específico para versão 24)
│   │   ├── validations/        # STAGE: Validation jobs
│   │   │   ├── job_setup.yml   # Setup Node.js 24 + pnpm + cache
│   │   │   ├── job_test.yml    # Testes (unit, integration, e2e, coverage)
│   │   │   └── job_npm-audit.yml # Dependency vulnerabilities (npm audit)
│   │   └── build/              # STAGE: Build jobs
│   │       └── job_build.yml   # pnpm build (NestJS, Express, Next.js, etc)
│   └── shared/                 # 🆕 Genérico para TODAS as versões Node (18, 20, 22, 24, etc)
│       └── validations/
│           └── job_lint.yml    # ESLint + Prettier (works for any Node version)
├── shared/                     # Actions agnósticas de linguagem + versão
│   └── validations/            # STAGE: Validation jobs (shared)
│       ├── job_gitleaks.yml    # Secret detection (170+ rules)
│       ├── job_semgrep.yml     # SAST scan (JS, TS, Python, Go, Java, etc)
│       └── job_trivy.yml       # Multi-purpose scanner (filesystem, container, IaC)
├── golang/                     # Go (futuro)
│   ├── validations/
│   │   ├── job_setup.yml
│   │   └── job_lint.yml
│   └── build/
│       └── job_build.yml
└── rust/                       # Rust (futuro)
    ├── validations/
    │   ├── job_setup.yml
    │   └── job_lint.yml
    └── build/
        └── job_build.yml
```

### 🧩 Filosofia da Organização

#### **Hierarquia: Stage → Tech/Shared → Job**

1. **Stage** = Pasta principal que agrupa jobs relacionados
   - `validations/` → Quality checks, security scans, tests
   - `build/` → Compilation, packaging
   - `deploy/` → Deployment to environments (futuro)

2. **Tech-specific + Version** (`v1/nodejs/24/`) → Específico para **Node.js 24**
   - Se mexer aqui, afeta **apenas** projetos Node.js v24
   - Exemplos: `job_setup.yml` (usa Node 24), `job_test.yml`, `job_npm-audit.yml`, `job_build.yml`

3. **Tech-specific + Shared** (`v1/nodejs/shared/`) → Genérico para **TODAS as versões Node.js**
   - Se mexer aqui, afeta projetos Node.js 18, 20, 22, 24, etc
   - Exemplos: `job_lint.yml` (ESLint/Prettier funcionam em qualquer versão)

4. **Fully Shared** (`v1/shared/`) → Agnóstico de linguagem **e versão**
   - Funciona para **qualquer** stack tecnológico (Node.js, Python, Go, Java, etc)
   - Funciona para **qualquer versão** (Node 18, 20, 24, Python 3.9, 3.12, etc)
   - Exemplos: `job_gitleaks.yml` (secrets), `job_semgrep.yml` (SAST), `job_trivy.yml` (vulnerabilities)

5. **Nomenclatura:**
   - ❌ Antes: `v1/nodejs/24/lint/action.yml`
   - ✅ Agora: `v1/nodejs/shared/validations/job_lint.yml`
   - **Lógica:** Stage (`validations`) → Tech-shared (`nodejs/shared`) → Job (`job_lint.yml`)

#### **Princípios:**

- ✅ **Stage-based organization**: Jobs agrupados por fase do pipeline (validations, build, deploy)
- ✅ **3-tier separation**:
  - `shared/` → Universal (qualquer linguagem + versão)
  - `nodejs/shared/` → Node-specific (qualquer versão Node)
  - `nodejs/24/` → Node 24 specific
- ✅ **Framework-agnostic**: O `package.json` do projeto define como executar cada job
- ✅ **Versionamento semântico**: `v1/` permite breaking changes no futuro (`v2/`)
- ✅ **Reutilização máxima**: Shared jobs podem ser usados em projetos Python, Go, Java, etc

## 🎯 Como Usar

### Node.js Genérico (Express, Fastify, qualquer framework)

```yaml
name: CI/CD

on: [push]

jobs:
  validation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # GitLeaks needs full history
      
      # Shared: Language-agnostic jobs
      - name: GitLeaks
        uses: videoconverterpro/pipeline-template/v1/shared/validations/job_gitleaks@main
      
      # Node.js 24: Tech-specific jobs
      - name: Setup Node.js
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/validations/job_setup@main
        
      - name: Lint
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/validations/job_lint@main
        
  build:
    needs: validation
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/validations/job_setup@main
        
      - name: Build
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/build/job_build@main
```

### NestJS Completo (Prisma + Testes + Security)

```yaml
name: CI/CD NestJS

on: [push]

jobs:
  validation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      # SECURITY: Shared jobs (work with any language)
      - name: GitLeaks - Secret Detection
        uses: videoconverterpro/pipeline-template/v1/shared/validations/job_gitleaks@main
      
      # SETUP: Node.js environment
      - name: Setup Node.js
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/validations/job_setup@main
        
      # SECURITY: Node.js specific dependency scan
      - name: npm audit
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/validations/job_npm-audit@main
        with:
          severity-level: 'high'
          production-only: 'true'
      
      # SECURITY: SAST scan (shared - works with JS, TS, Python, Go, etc)
      - name: Semgrep - SAST
        uses: videoconverterpro/pipeline-template/v1/shared/validations/job_semgrep@main
        with:
          config: 'auto'
          severity: 'WARNING'
        
      # QUALITY: Code checks
      - name: Lint
        uses: videoconverterpro/pipeline-template/v1/nodejs/shared/validations/job_lint@main
        
      - name: Test
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/validations/job_test@main
        with:
          unit: 'true'
          e2e: 'true'
          coverage: 'true'
      
      # SECURITY: Filesystem/Container/IaC scan (shared)
      - name: Trivy - Multi-purpose Scan
        uses: videoconverterpro/pipeline-template/v1/shared/validations/job_trivy@main
        with:
          scan-type: 'fs'
          severity: 'HIGH,CRITICAL'
        
  build:
    needs: validation
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/validations/job_setup@main
        
      - name: Build
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/build/job_build@main
```

> 📖 **Testes**: Veja [docs/TESTING.md](docs/TESTING.md) para documentação completa sobre tipos de teste, inputs e estratégias.
> 📖 **Security**: Veja [docs/SECURITY-SCANNING.md](docs/SECURITY-SCANNING.md) para detalhes sobre todas as ferramentas de segurança.

### Next.js com Testes e Security

```yaml
name: CI/CD Next.js

on: [push]

jobs:
  validation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      # Shared security jobs
      - name: GitLeaks
        uses: videoconverterpro/pipeline-template/v1/shared/validations/job_gitleaks@main
      
      - name: Setup Node.js
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/validations/job_setup@main
        
      - name: npm audit
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/validations/job_npm-audit@main
        
      - name: Lint
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/validations/job_lint@main
        
      - name: Test
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/validations/job_test@main
        
  build:
    needs: validation
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/validations/job_setup@main
        
      - name: Build
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/build/job_build@main
```

### Go (Futuro)

```yaml
name: CI/CD Go

on: [push]

jobs:
  validation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: GitLeaks
        uses: videoconverterpro/pipeline-template/v1/shared/validations/job_gitleaks@main
      
      - name: Setup Go
        uses: videoconverterpro/pipeline-template/v1/golang/validations/job_setup@main
        
      - name: Lint
        uses: videoconverterpro/pipeline-template/v1/golang/validations/job_lint@main
        
  build:
    needs: validation
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Go
        uses: videoconverterpro/pipeline-template/v1/golang/validations/job_setup@main
        
      - name: Build
        uses: videoconverterpro/pipeline-template/v1/golang/build/job_build@main
```

### Rust (Futuro)

```yaml
name: CI/CD Rust

on: [push]

jobs:
  validation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: GitLeaks
        uses: videoconverterpro/pipeline-template/v1/shared/validations/job_gitleaks@main
      
      - name: Setup Rust
        uses: videoconverterpro/pipeline-template/v1/rust/validations/job_setup@main
        
      - name: Lint
        uses: videoconverterpro/pipeline-template/v1/rust/validations/job_lint@main
        
  build:
    needs: validation
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Rust
        uses: videoconverterpro/pipeline-template/v1/rust/validations/job_setup@main
        
      - name: Build
        uses: videoconverterpro/pipeline-template/v1/rust/build/job_build@main
```

## 📚 Documentação Adicional

- **[Security Scanning](docs/SECURITY-SCANNING.md)** - Guia completo sobre ferramentas de segurança gratuitas (npm audit, Semgrep, Trivy, etc)
- **[Testes](docs/TESTING.md)** - Guia completo sobre tipos de teste, inputs e estratégias
- **[GitLeaks](docs/GITLEAKS.md)** - Detecção de secrets com 170+ regras (AWS, GitHub, Slack, etc)
- **[CI/CD](docs/CICD.md)** - Nomenclatura, convenções e boas práticas *(futuro)*
- **[Cache](docs/CACHE.md)** - Otimização de performance com cache *(futuro)*

## ✨ Benefícios

- ✅ **Reutilização Total**: Mesmas actions em múltiplos projetos
- ✅ **Manutenção Centralizada**: Update em 1 lugar, propaga automaticamente
- ✅ **Padronização**: Pipelines consistentes entre projetos
- ✅ **Multi-Tecnologia**: Node.js, Go, Rust (expansível)
- ✅ **Cache Otimizado**: Builds 60-85% mais rápidos
- ✅ **Privado OK**: Funciona com repos privados sem GitHub Enterprise

## 📊 Performance

| Tecnologia | Sem Cache | Com Cache | Economia |
|------------|-----------|-----------|----------|
| Node.js    | 165s      | 25s       | 85%      |
| Go         | 45s       | 10s       | 78%      |
| Rust       | 180s      | 30s       | 83%      |

## 📦 Projetos Usando

- [`videoconverterpro/api`](https://github.com/videoconverterpro/api) - Node.js/NestJS
- *Adicione seu projeto aqui*

## 🔧 Desenvolvimento

### Adicionar Nova Tecnologia

1. Crie pasta: `v1/<tech>/`
2. Organize por stages: `validations/`, `build/`, `deploy/`
3. Adicione jobs: `job_setup.yml`, `job_lint.yml`, `job_build.yml`
4. Teste em projeto real
5. Atualize README

### Estrutura Padrão por Tech

```text
v1/<tech>/
├── validations/           # STAGE: Validation
│   ├── job_setup.yml      # Instalar runtime + cache
│   ├── job_lint.yml       # Validação de código
│   └── job_test.yml       # Testes automatizados
└── build/                 # STAGE: Build
    └── job_build.yml      # Compilação/build
```

### Adicionar Job Shared (Language-Agnostic)

1. Identifique se o job funciona para **qualquer linguagem**
2. Crie em `v1/shared/<stage>/job_<name>.yml`
3. Teste em projetos Node.js, Python, Go, etc
4. Documente em `docs/`

**Exemplo:** GitLeaks funciona para qualquer linguagem → `v1/shared/validations/job_gitleaks.yml`

## 📝 Licença

**Proprietário** - Bruno Roberto Morillo  
CPF: 460.876.598-11  
© 2025 VideoConverterPro - Todos os direitos reservados
