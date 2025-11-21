# 🚀 Pipeline Template - CI/CD Reutilizável Multi-Tech

[![License](https://img.shields.io/badge/License-Proprietary-red.svg)](LICENSE)
[![Node.js](https://img.shields.io/badge/Node.js-24+-green.svg)](https://nodejs.org/)
[![Go](https://img.shields.io/badge/Go-1.22+-00ADD8.svg)](https://go.dev/)
[![Rust](https://img.shields.io/badge/Rust-stable-orange.svg)](https://rust-lang.org/)

Repositório centralizado de **composite actions** para pipelines CI/CD em múltiplas tecnologias.

## 📁 Estrutura

```
v1/
├── nodejs/24/              # Node.js 24 (genérico para qualquer framework)
│   ├── setup/              # Setup Node.js + pnpm + cache
│   ├── lint/               # Prettier + ESLint (framework-agnostic)
│   └── nestjs/             # Actions específicas do NestJS
│       ├── build/          # Build com Prisma + validações
│       └── test/           # Testes unitários + e2e + coverage
├── golang/                 # Go (futuro)
│   ├── setup/
│   ├── lint/
│   └── build/
└── rust/                   # Rust (futuro)
    ├── setup/
    ├── lint/
    └── build/
```

### 🧩 Filosofia da Organização

- **Genérico primeiro**: Actions em `v1/nodejs/24/` funcionam para **qualquer projeto Node.js**
- **Específico quando necessário**: Subpastas por framework (`nestjs/`, `express/`, `nextjs/`) apenas para steps únicos
- **Versionamento semântico**: `v1/` permite breaking changes no futuro (`v2/` sem quebrar projetos antigos)

## 🎯 Como Usar

### Node.js Genérico (Express, Fastify, qualquer framework)

```yaml
name: CI/CD

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js + pnpm
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/setup@main
        
      - name: Lint (Prettier + ESLint)
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/lint@main
        
      - name: Build genérico
        run: pnpm build
```

### NestJS Completo (com Prisma + testes)

```yaml
name: CI/CD NestJS

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js + pnpm
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/setup@main
        
      - name: Lint (Prettier + ESLint)
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/lint@main
        
      - name: Build NestJS com Prisma
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/nestjs/build@main
      
      - name: Testes (unitários + e2e)
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/nestjs/test@main
        with:
          run-e2e: 'true'
          coverage: 'true'
```
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Rust
        uses: videoconverterpro/pipeline-template/.github/actions/rust/setup@main
        with:
          rust-version: 'stable'
      
      - name: Lint (clippy + rustfmt)
        uses: videoconverterpro/pipeline-template/.github/actions/rust/lint@main
      
      - name: Build
        uses: videoconverterpro/pipeline-template/.github/actions/rust/build@main
        with:
          profile: 'release'
```

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
- _Adicione seu projeto aqui_

## 🔧 Desenvolvimento

### Adicionar Nova Tecnologia

1. Crie pasta: `.github/actions/<tech>/`
2. Adicione 3 actions: `setup/`, `lint/`, `build/`
3. Teste em projeto real
4. Atualize README

### Estrutura Padrão

```
<tech>/
├── setup/action.yml      # Instalar runtime + cache
├── lint/action.yml       # Validação de código
└── build/action.yml      # Compilação/build
```

## 📝 Licença

**Proprietário** - Bruno Roberto Morillo  
CPF: 460.876.598-11  
© 2025 VideoConverterPro - Todos os direitos reservados
