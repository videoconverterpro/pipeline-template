# 🔒 GitLeaks - Secret Detection

Documentação completa sobre a action de detecção de secrets usando GitLeaks.

## 📋 Visão Geral

A action `v1/nodejs/24/gitleaks` escaneia todo o repositório Git (histórico completo) em busca de **secrets acidentalmente commitados** como:

- 🔑 Chaves API (AWS, Azure, GCP, Stripe, SendGrid, etc.)
- 🎫 Tokens de acesso (GitHub, GitLab, Slack, Discord, etc.)
- 🔐 Senhas e credentials
- 📜 Certificados e chaves privadas
- 🗄️ Connection strings de banco de dados
- 🌐 URLs com credenciais embutidas

## ⚡ Por Que Usar GitLeaks?

**Problema:** Secrets commitados no Git **NUNCA podem ser completamente removidos** do histórico (mesmo após rebase/force push, podem estar em forks, backups, CI cache, etc.)

**Solução:** Detectar secrets **ANTES** de merge para produção.

## 🔧 Inputs Disponíveis

| Input | Tipo | Padrão | Descrição |
|-------|------|--------|-----------|
| `gitleaks-version` | string | `v8.21.2` | Versão do GitLeaks (imagem Docker) |
| `continue-on-error` | boolean | `false` | Continuar pipeline mesmo se secrets forem encontrados |
| `show-all-rules` | boolean | `false` | Exibir todas as 170+ regras antes do scan |

## 📖 Exemplos de Uso

### Exemplo 1: Básico (recomendado)

```yaml
- name: "[STEP] GitLeaks"
  uses: videoconverterpro/pipeline-template/v1/nodejs/24/gitleaks@main
```

**Comportamento:**
- ✅ Falha se secrets forem encontrados
- ✅ Gera artifact com relatório JSON
- ✅ Exibe resumo das findings

### Exemplo 2: Com visualização de regras

```yaml
- name: "[STEP] GitLeaks"
  uses: videoconverterpro/pipeline-template/v1/nodejs/24/gitleaks@main
  with:
    show-all-rules: 'true'
```

**Uso:** Primeira execução ou debugging - veja todas as 170+ regras aplicadas

### Exemplo 3: Continue on error (não recomendado para produção)

```yaml
- name: "[STEP] GitLeaks"
  uses: videoconverterpro/pipeline-template/v1/nodejs/24/gitleaks@main
  with:
    continue-on-error: 'true'
```

**Uso:** Migração gradual - permite merge mas gera warnings

### Exemplo 4: Versão específica

```yaml
- name: "[STEP] GitLeaks"
  uses: videoconverterpro/pipeline-template/v1/nodejs/24/gitleaks@main
  with:
    gitleaks-version: 'v8.18.0'
```

## 🎯 Integração no Pipeline

### Recomendado: Stage Validation

```yaml
jobs:
  validation:
    name: "[STAGE] Validation"
    runs-on: ubuntu-latest
    
    steps:
      - name: "[STEP] Checkout"
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # IMPORTANTE: histórico completo
      
      - name: "[STEP] GitLeaks"
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/gitleaks@main
      
      - name: "[STEP] Setup"
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/setup@main
      
      - name: "[STEP] Lint"
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/lint@main
```

**⚠️ IMPORTANTE:** `fetch-depth: 0` é necessário para escanear todo o histórico Git.

### Alternativa: Job separado (paralelização)

```yaml
jobs:
  security:
    name: "[STAGE] Security Scan"
    runs-on: ubuntu-latest
    
    steps:
      - name: "[STEP] Checkout"
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: "[STEP] GitLeaks"
        uses: videoconverterpro/pipeline-template/v1/nodejs/24/gitleaks@main
  
  validation:
    name: "[STAGE] Validation"
    runs-on: ubuntu-latest
    needs: security  # Aguarda security scan
    
    steps:
      # ... lint, test, etc
```

## 🛡️ Regras de Detecção

GitLeaks v8.21.2 inclui **170+ regras padrão** para detectar:

### Categorias Principais

#### 1. Cloud Providers
- AWS Access Key, Secret Key, Session Token
- Azure Storage Key, SAS Token, Service Principal
- GCP API Key, Service Account, OAuth Token
- DigitalOcean Token
- Heroku API Key

#### 2. Version Control
- GitHub Token (ghp_, gho_, ghs_, ghu_)
- GitLab Token
- Bitbucket Token

#### 3. Communication Platforms
- Slack Token (xoxb-, xoxp-, xoxa-, xoxr-)
- Discord Token
- Telegram Bot Token

#### 4. Payment & Services
- Stripe API Key
- PayPal Token
- Square Access Token
- Twilio API Key
- SendGrid API Key

#### 5. Databases
- MongoDB Connection String
- PostgreSQL Connection String
- MySQL Connection String
- Redis URL

#### 6. Generic Patterns
- API Keys (32-64 caracteres alfanuméricos)
- Passwords em variáveis
- Private Keys (RSA, SSH, PGP)
- Bearer Tokens
- JWT Tokens

### Regras Customizadas

Além das 170+ padrão, a action inclui 2 regras adicionais:

```toml
[[rules]]
id = "generic-api-key"
regex = '''(?i)(api[_-]?key|apikey)[\s]*[=:]+[\s]*['"]?([a-zA-Z0-9_\-]{32,})['"]?'''

[[rules]]
id = "generic-secret"
regex = '''(?i)(secret|password|passwd|pwd|token)[\s]*[=:]+[\s]*['"]?([a-zA-Z0-9_\-!@#$%^&*()]{8,})['"]?'''
```

## 🔍 Como Ver Todas as Regras

### Método 1: Durante o scan (recomendado)

```yaml
- uses: videoconverterpro/pipeline-template/v1/nodejs/24/gitleaks@main
  with:
    show-all-rules: 'true'
```

### Método 2: Localmente via Docker

```bash
docker run --rm zricethezav/gitleaks:v8.21.2 version
docker run --rm zricethezav/gitleaks:v8.21.2 rules list
```

### Método 3: Documentação Oficial

Todas as regras padrão: https://github.com/gitleaks/gitleaks/blob/master/config/gitleaks.toml

## 📝 Configuração Personalizada

### Criar .gitleaks.toml no repositório

```toml
title = "My Custom GitLeaks Config"

# Adicionar regras customizadas
[[rules]]
id = "my-custom-token"
description = "My Custom API Token"
regex = '''MY_TOKEN_[A-Z0-9]{32}'''

# Arquivos/pastas ignorados
[allowlist]
paths = [
  '''\.git/''',
  '''node_modules/''',
  '''test/fixtures/''',
]

# Ignorar commits específicos
commitsSince = "2024-01-01"
```

### Criar .gitleaksignore no repositório

Para ignorar findings específicos (após rotacionar secrets):

```
# Formato: <fingerprint-do-finding>
# Obtenha o fingerprint do relatório gitleaks-report.json

a1b2c3d4e5f6g7h8i9j0
9876543210abcdefghij
```

## 🚨 O Que Fazer Quando Secrets São Encontrados

### 1. ⚠️ ROTATE IMEDIATAMENTE

```bash
# Exemplo: Rotacionar AWS keys
aws iam delete-access-key --access-key-id AKIAIOSFODNN7EXAMPLE
aws iam create-access-key --user-name myuser

# Exemplo: Regenerar GitHub token
# Acesse: Settings → Developer settings → Personal access tokens → Regenerate
```

### 2. 📥 Baixar o Relatório

No GitHub Actions:
1. Acesse a run que falhou
2. Vá em "Summary"
3. Baixe o artifact `gitleaks-report-<sha>`
4. Abra `gitleaks-report.json`

### 3. 🔍 Identificar Commits Afetados

```bash
# Ver commit específico
git show <commit-hash>

# Ver histórico do arquivo
git log --follow -- path/to/file
```

### 4. 🛠️ Criar .gitleaksignore

Após rotacionar o secret, adicione o fingerprint:

```bash
# Extrair fingerprints do relatório
jq -r '.[].Fingerprint' gitleaks-report.json > .gitleaksignore

# Commitar o arquivo
git add .gitleaksignore
git commit -m "chore: adicionar gitleaksignore após rotação de secrets"
```

### 5. ✅ Re-executar Pipeline

Após rotacionar + ignorar, o pipeline deve passar.

## 🎭 False Positives

GitLeaks pode detectar padrões que **parecem** secrets mas não são:

### Exemplos Comuns

```typescript
// ❌ Detectado como secret
const DEMO_API_KEY = "sk_test_1234567890abcdefghijklmnopqrstuv";

// ✅ Solução 1: Usar .gitleaksignore
// ✅ Solução 2: Usar variáveis de ambiente
const DEMO_API_KEY = process.env.DEMO_KEY;

// ✅ Solução 3: Comentário especial
const DEMO_KEY = "sk_test_demo"; // gitleaks:allow
```

### Reduzir False Positives

```toml
# .gitleaks.toml
[allowlist]
description = "Allowlist test fixtures"
paths = [
  '''test/fixtures/''',
  '''__mocks__/''',
  '''examples/''',
]

regexes = [
  '''sk_test_''',  # Test keys do Stripe
  '''demo_''',     # Chaves de demonstração
]
```

## 📊 Relatório JSON

Estrutura do `gitleaks-report.json`:

```json
[
  {
    "Description": "AWS Access Key",
    "StartLine": 42,
    "EndLine": 42,
    "StartColumn": 15,
    "EndColumn": 35,
    "Match": "AKIAIOSFODNN7EXAMPLE",
    "Secret": "AKIAIOSFODNN7EXAMPLE",
    "File": "src/config/aws.ts",
    "Commit": "a1b2c3d4e5f6g7h8i9j0",
    "Author": "developer@example.com",
    "Date": "2024-11-20T10:30:00Z",
    "Message": "feat: add AWS integration",
    "RuleID": "aws-access-token",
    "Fingerprint": "a1b2c3d4e5f6g7h8i9j0:src/config/aws.ts:aws-access-token:42"
  }
]
```

## 🔗 Integração com Outras Ferramentas

### GitHub Security (Advanced Security)

```yaml
- name: "[STEP] GitLeaks"
  uses: videoconverterpro/pipeline-template/v1/nodejs/24/gitleaks@main

- name: Upload to GitHub Security
  if: always()
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: gitleaks.sarif
```

### Notificação no Slack

```yaml
- name: "[STEP] GitLeaks"
  id: gitleaks
  uses: videoconverterpro/pipeline-template/v1/nodejs/24/gitleaks@main
  continue-on-error: true

- name: Notify Slack
  if: env.GITLEAKS_FOUND > 0
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "⚠️ GitLeaks found ${{ env.GITLEAKS_FOUND }} secrets!",
        "blocks": [...]
      }
```

## 🎯 Best Practices

### ✅ DO

1. **Execute no stage de validation** (antes de build/deploy)
2. **Use `fetch-depth: 0`** para escanear histórico completo
3. **Rotacione secrets imediatamente** quando detectados
4. **Documente no .gitleaksignore** após rotação
5. **Revise periodicamente** as regras customizadas

### ❌ DON'T

1. **Não use `continue-on-error: true`** em produção
2. **Não commite .gitleaksignore sem rotacionar** os secrets
3. **Não ignore o histórico Git** (fetch-depth: 1)
4. **Não desabilite GitLeaks** "temporariamente"
5. **Não compartilhe relatórios publicamente** (podem conter secrets)

## 📚 Recursos Adicionais

- **GitLeaks GitHub**: https://github.com/gitleaks/gitleaks
- **Documentação Oficial**: https://github.com/gitleaks/gitleaks/wiki
- **Regras Padrão**: https://github.com/gitleaks/gitleaks/blob/master/config/gitleaks.toml
- **Releases**: https://github.com/gitleaks/gitleaks/releases

## 📝 Licença

**Proprietário** - Bruno Roberto Morillo  
CPF: 460.876.598-11  
© 2025 VideoConverterPro - Todos os direitos reservados
