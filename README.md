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
├── nodejs/24/              # Node.js 24 (genérico para qualquer framework)
│   ├── setup/              # Setup Node.js + pnpm + cache
│   ├── lint/               # Prettier + ESLint
│   ├── test/               # Testes (unit, integration, e2e, coverage)
│   └── build/              # pnpm build (funciona com NestJS, Express, Next.js, etc)
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

- **100% Genérico**: Actions funcionam para **qualquer projeto Node.js** (NestJS, Express, Next.js, etc)
- **Framework-agnostic**: O `package.json` do projeto define como executar `build`, `lint`, `test`
- **Versionamento semântico**: `v1/` permite breaking changes no futuro (`v2/` sem quebrar projetos antigos)
- **Simplicidade**: Evitamos subdivisões por framework - mantém o template enxuto e manutenível

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
        
      - name: Build
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/build@main
```

### NestJS Completo (Prisma + Testes)

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
        
      - name: Lint
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/lint@main
        
      - name: Test
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/test@main
        with:
          unit: 'true'
          e2e: 'true'
        
      - name: Build
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/build@main
```

> 📖 **Testes**: Veja [docs/TESTING.md](docs/TESTING.md) para documentação completa sobre tipos de teste, inputs e estratégias.

### Next.js com Testes

```yaml
name: CI/CD Next.js

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js + pnpm
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/setup@main
        
      - name: Lint
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/lint@main
        
      - name: Test
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/test@main
        
      - name: Build
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/build@main
```

### Go (Futuro)

```yaml
name: CI/CD Go

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Go
        uses: videoconverterpro/pipeline-template/v1/golang/setup@main
        
      - name: Lint
        uses: videoconverterpro/pipeline-template/v1/golang/lint@main
        
      - name: Build
        uses: videoconverterpro/pipeline-template/v1/golang/build@main
```

### Rust (Futuro)

```yaml
name: CI/CD Rust

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Rust
        uses: videoconverterpro/pipeline-template/v1/rust/setup@main
        
      - name: Lint
        uses: videoconverterpro/pipeline-template/v1/rust/lint@main
        
      - name: Build
        uses: videoconverterpro/pipeline-template/v1/rust/build@main
```

## 📚 Documentação Adicional

- **[Testes](docs/TESTING.md)** - Guia completo sobre tipos de teste, inputs e estratégias
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
2. Adicione 3 actions: `setup/`, `lint/`, `build/`
3. Teste em projeto real
4. Atualize README

### Estrutura Padrão

```text
v1/<tech>/
├── setup/action.yml      # Instalar runtime + cache
├── lint/action.yml       # Validação de código
└── build/action.yml      # Compilação/build
```

## 📝 Licença

**Proprietário** - Bruno Roberto Morillo  
CPF: 460.876.598-11  
© 2025 VideoConverterPro - Todos os direitos reservados
