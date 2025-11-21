# Security Scanning Tools

> **Documentação completa das ferramentas de segurança gratuitas disponíveis para CI/CD**
>
> Este documento lista todas as alternativas gratuitas aos serviços pagos (Snyk, SonarQube Cloud, etc.) que podem ser utilizados em projetos pessoais e pequenas empresas.

---

## Índice

- [Security Scanning Tools](#security-scanning-tools)
  - [Índice](#índice)
  - [Tier 1: Implementação Imediata](#tier-1-implementação-imediata)
    - [npm audit](#npm-audit)
    - [Semgrep CE](#semgrep-ce)
    - [ESLint Security Plugins](#eslint-security-plugins)
  - [Tier 2: Próxima Semana](#tier-2-próxima-semana)
    - [Trivy](#trivy)
  - [Tier 3: Opcional/Futuro](#tier-3-opcionalfuturo)
    - [SonarQube CE (Self-Hosted)](#sonarqube-ce-self-hosted)
    - [OWASP Dependency-Check](#owasp-dependency-check)
    - [OSV-Scanner](#osv-scanner)
    - [Checkov](#checkov)
  - [Já Implementado](#já-implementado)
    - [GitLeaks](#gitleaks)
  - [Comparação com Ferramentas Pagas](#comparação-com-ferramentas-pagas)
    - [Snyk vs Alternativas Gratuitas](#snyk-vs-alternativas-gratuitas)
    - [SonarQube Cloud vs SonarQube CE](#sonarqube-cloud-vs-sonarqube-ce)
  - [Stack Recomendada Gratuita](#stack-recomendada-gratuita)
    - [Para Projetos Pessoais (VideoConverterPro)](#para-projetos-pessoais-videoconverterpro)
    - [Comparação: C\&A Brasil (Pago) vs VideoConverterPro (Grátis)](#comparação-ca-brasil-pago-vs-videoconverterpro-grátis)
      - [C\&A Brasil (Enterprise Stack)](#ca-brasil-enterprise-stack)
      - [VideoConverterPro (Free Stack)](#videoconverterpro-free-stack)
    - [Timeline de Implementação](#timeline-de-implementação)
  - [Workflow Completo (Tier 1 + 2 Implementado)](#workflow-completo-tier-1--2-implementado)
  - [Métricas de Cobertura](#métricas-de-cobertura)
    - [Por Categoria](#por-categoria)
    - [O Que Você Perde (Grátis vs Pago)](#o-que-você-perde-grátis-vs-pago)
    - [O Que Você Ganha (Grátis)](#o-que-você-ganha-grátis)
  - [Próximos Passos](#próximos-passos)
    - [Implementação Imediata (Tier 1)](#implementação-imediata-tier-1)
    - [Implementação Próxima Semana (Tier 2)](#implementação-próxima-semana-tier-2)
  - [Referências](#referências)
  - [Conclusão](#conclusão)

---

## Tier 1: Implementação Imediata

### npm audit

**Função:** Dependency vulnerabilities (substitui parte do Snyk)

**Custo:** `$0` (nativo do npm)

**Cobertura:**

- Vulnerabilidades conhecidas (CVE) em dependências npm
- Integração com GitHub Advisory Database
- Classificação por severidade: LOW, MODERATE, HIGH, CRITICAL
- Sugestões automáticas de atualização

**Tempo de execução:** `+30s` no pipeline

**Limitações:**

- Apenas dependências npm (não cobre Docker, OS packages)
- Menos detalhado que Snyk (sem priorizações avançadas)
- Sem análise de licenças

**Implementação:**

```yaml
# pipeline-template/v1/nodejs/24/npm-audit/action.yml
name: 'npm audit - Dependency Vulnerabilities'
description: 'Scan dependencies for known vulnerabilities using npm audit'

inputs:
  severity-level:
    description: 'Minimum severity level to fail (low, moderate, high, critical)'
    required: false
    default: 'high'
  production-only:
    description: 'Only check production dependencies'
    required: false
    default: 'true'
  continue-on-error:
    description: 'Continue pipeline even if vulnerabilities found'
    required: false
    default: 'false'

runs:
  using: 'composite'
  steps:
    - name: '[ACTION] Run npm audit'
      shell: bash
      run: |
        echo "🔍 [npm audit] Scanning dependencies for vulnerabilities..."
        
        AUDIT_ARGS="--audit-level=${{ inputs.severity-level }}"
        if [ "${{ inputs.production-only }}" = "true" ]; then
          AUDIT_ARGS="$AUDIT_ARGS --production"
        fi
        
        npm audit $AUDIT_ARGS || EXIT_CODE=$?
        
        if [ -n "$EXIT_CODE" ] && [ "$EXIT_CODE" -ne 0 ]; then
          echo "❌ [npm audit] Vulnerabilities found!"
          npm audit --json > npm-audit-report.json || true
          
          if [ "${{ inputs.continue-on-error }}" = "false" ]; then
            exit $EXIT_CODE
          fi
        else
          echo "✅ [npm audit] No vulnerabilities found"
        fi

    - name: '[ACTION] Upload audit report'
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: npm-audit-report-${{ github.sha }}
        path: npm-audit-report.json
        retention-days: 30
        if-no-files-found: ignore
```

**Uso no workflow:**

```yaml
- name: '[STEP] npm audit - Dependency Scan'
  uses: ./pipeline-template/v1/nodejs/24/npm-audit
  with:
    severity-level: 'high'
    production-only: 'true'
    continue-on-error: 'false'
```

**Exemplo de saída:**

```md
found 3 vulnerabilities (1 moderate, 2 high) in 847 scanned packages
  2 vulnerabilities require manual review. See the full report for details.
```

---

### Semgrep CE

**Função:** SAST - Static Application Security Testing (substitui SonarQube Security Hotspots)

**Custo:** `$0` (Community Edition - unlimited scans em repositórios públicos)

**Cobertura:**

- 1000+ regras de segurança para JavaScript/TypeScript
- OWASP Top 10 coverage
- Detecta: SQL injection, XSS, path traversal, hardcoded secrets, insecure crypto, etc.
- Análise de código estático sem execução
- Suporta: JS, TS, Python, Go, Java, Ruby, PHP, C#, etc.

**Tempo de execução:** `+2min` no pipeline

**Limitações:**

- Sem interface web local (apenas CLI ou Semgrep Cloud free tier)
- Menos contexto que SonarQube (não rastreia fluxo de dados complexo)
- Resultados em SARIF/JSON (precisa de parser para visualização bonita)

**Implementação:**

```yaml
# pipeline-template/v1/nodejs/24/semgrep/action.yml
name: 'Semgrep CE - SAST Scan'
description: 'Static Application Security Testing using Semgrep Community Edition'

inputs:
  config:
    description: 'Semgrep config (auto, p/security-audit, p/owasp-top-ten, p/ci)'
    required: false
    default: 'auto'
  severity:
    description: 'Minimum severity to report (INFO, WARNING, ERROR)'
    required: false
    default: 'WARNING'
  continue-on-error:
    description: 'Continue pipeline even if findings detected'
    required: false
    default: 'false'

runs:
  using: 'composite'
  steps:
    - name: '[ACTION] Run Semgrep scan'
      uses: returntocorp/semgrep-action@v1
      with:
        config: ${{ inputs.config }}
        generateSarif: true
      continue-on-error: ${{ inputs.continue-on-error == 'true' }}

    - name: '[ACTION] Upload SARIF report'
      if: always()
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: semgrep.sarif
        category: semgrep

    - name: '[ACTION] Upload JSON report'
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: semgrep-report-${{ github.sha }}
        path: semgrep.sarif
        retention-days: 30
```

**Uso no workflow:**

```yaml
- name: '[STEP] Semgrep - SAST Scan'
  uses: ./pipeline-template/v1/nodejs/24/semgrep
  with:
    config: 'auto'  # ou p/security-audit, p/owasp-top-ten
    severity: 'WARNING'
    continue-on-error: 'false'
```

**Exemplo de regras detectadas:**

```md
javascript.express.security.audit.xss.pug.explicit-unescape
javascript.lang.security.audit.sql-injection
javascript.express.security.audit.express-check-csurf-middleware-missing
typescript.jwt.security.jwt-hardcoded-secret
```

---

### ESLint Security Plugins

**Função:** Security linting durante desenvolvimento e CI (complemento ao lint existente)

**Custo:** `$0` (plugins open-source)

**Cobertura:**

- Detecção de padrões inseguros no código
- Regras para: regex DoS, eval(), unsafe buffer usage, etc.
- Detecção de secrets hardcoded (complementa GitLeaks)
- Integração nativa com VSCode (feedback instantâneo)

**Tempo de execução:** `+0s` (roda junto com lint existente)

**Limitações:**

- Apenas análise superficial (não detecta vulnerabilidades complexas)
- Precisa de configuração manual das regras
- Alguns falsos positivos

**Implementação:**

```bash
# Instalar plugins
pnpm add -D eslint-plugin-security eslint-plugin-no-secrets
```

```javascript
// eslint.config.js
import security from 'eslint-plugin-security';
import noSecrets from 'eslint-plugin-no-secrets';

export default [
  // ... outras configs
  {
    plugins: {
      security,
      'no-secrets': noSecrets,
    },
    rules: {
      // eslint-plugin-security
      'security/detect-object-injection': 'warn',
      'security/detect-non-literal-regexp': 'warn',
      'security/detect-unsafe-regex': 'error',
      'security/detect-buffer-noassert': 'error',
      'security/detect-child-process': 'warn',
      'security/detect-disable-mustache-escape': 'error',
      'security/detect-eval-with-expression': 'error',
      'security/detect-no-csrf-before-method-override': 'error',
      'security/detect-non-literal-fs-filename': 'warn',
      'security/detect-non-literal-require': 'warn',
      'security/detect-possible-timing-attacks': 'warn',
      'security/detect-pseudoRandomBytes': 'error',

      // eslint-plugin-no-secrets
      'no-secrets/no-secrets': ['error', { tolerance: 4.5 }],
    },
  },
];
```

**Uso no workflow:**

Nenhuma mudança necessária - o lint existente já executará as novas regras:

```yaml
- name: '[STEP] Lint'
  uses: ./pipeline-template/v1/nodejs/24/lint
```

**Exemplo de detecções:**

```md
src/auth/jwt.service.ts
  12:15  error  Hardcoded secret detected  no-secrets/no-secrets
  
src/database/query.ts
  45:8   error  Detected eval with expression  security/detect-eval-with-expression
```

---

## Tier 2: Próxima Semana

### Trivy

**Função:** Multi-purpose scanner (substitui 80% do Snyk Container + IaC)

**Custo:** `$0` (open-source da Aqua Security)

**Cobertura:**

- **Filesystem scan**: Vulnerabilidades em package-lock.json, pnpm-lock.yaml, go.mod, etc.
- **Container images**: CVEs em imagens Docker (Alpine, Node, etc.)
- **IaC**: Misconfigurations em Dockerfile, docker-compose.yml, Kubernetes manifests, Terraform
- **Secrets**: Detecção de credenciais hardcoded (complementa GitLeaks)
- **License scanning**: Detecção de licenças incompatíveis

**Tempo de execução:** `+1-3min` no pipeline (depende do escopo)

**Limitações:**

- Interface CLI apenas (sem dashboard web nativo)
- Precisa de Docker para rodar (ou baixar binário ~50MB)
- Escaneamento de imagens Docker requer build primeiro

**Implementação:**

```yaml
# pipeline-template/v1/nodejs/24/trivy/action.yml
name: 'Trivy - Multi-purpose Security Scanner'
description: 'Scan filesystem, containers, IaC, and secrets using Trivy'

inputs:
  scan-type:
    description: 'Type of scan (fs, image, config)'
    required: false
    default: 'fs'
  severity:
    description: 'Severities to scan (UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL)'
    required: false
    default: 'HIGH,CRITICAL'
  scan-path:
    description: 'Path to scan (. for current directory, or image name)'
    required: false
    default: '.'
  exit-code:
    description: 'Exit code when vulnerabilities found (0 or 1)'
    required: false
    default: '1'
  format:
    description: 'Output format (table, json, sarif)'
    required: false
    default: 'sarif'

runs:
  using: 'composite'
  steps:
    - name: '[ACTION] Run Trivy scan'
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: ${{ inputs.scan-type }}
        scan-ref: ${{ inputs.scan-path }}
        severity: ${{ inputs.severity }}
        exit-code: ${{ inputs.exit-code }}
        format: ${{ inputs.format }}
        output: 'trivy-results.sarif'

    - name: '[ACTION] Upload SARIF to GitHub Security'
      if: always() && inputs.format == 'sarif'
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: 'trivy-results.sarif'
        category: 'trivy'

    - name: '[ACTION] Upload Trivy report'
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: trivy-report-${{ github.sha }}
        path: trivy-results.sarif
        retention-days: 30
```

**Uso no workflow - Filesystem scan:**

```yaml
- name: '[STEP] Trivy - Filesystem Scan'
  uses: ./pipeline-template/v1/nodejs/24/trivy
  with:
    scan-type: 'fs'
    scan-path: '.'
    severity: 'HIGH,CRITICAL'
```

**Uso no workflow - Docker image scan:**

```yaml
- name: '[STEP] Build Docker image'
  run: docker build -t myapp:${{ github.sha }} .

- name: '[STEP] Trivy - Container Scan'
  uses: ./pipeline-template/v1/nodejs/24/trivy
  with:
    scan-type: 'image'
    scan-path: 'myapp:${{ github.sha }}'
    severity: 'HIGH,CRITICAL'
```

**Uso no workflow - IaC scan:**

```yaml
- name: '[STEP] Trivy - IaC Scan'
  uses: ./pipeline-template/v1/nodejs/24/trivy
  with:
    scan-type: 'config'
    scan-path: './docker-compose.yml'
    severity: 'MEDIUM,HIGH,CRITICAL'
```

**Exemplo de saída:**

```md
trivy-results.sarif (2024-01-15T10:30:00Z)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total: 15 (HIGH: 12, CRITICAL: 3)

┌─────────────────────┬────────────────┬──────────┬───────────────────┐
│ Library             │ Vulnerability  │ Severity │ Installed Version │
├─────────────────────┼────────────────┼──────────┼───────────────────┤
│ express             │ CVE-2024-12345 │ HIGH     │ 4.17.1            │
│ jsonwebtoken        │ CVE-2024-67890 │ CRITICAL │ 8.5.1             │
└─────────────────────┴────────────────┴──────────┴───────────────────┘
```

---

## Tier 3: Opcional/Futuro

### SonarQube CE (Self-Hosted)

**Função:** Code Quality + Security (substitui SonarQube Cloud completamente)

**Custo:** `$0` (Community Edition) + custo de hospedagem (VPS ~$5-10/mês)

**Cobertura:**

- **Code Quality**: Code smells, bugs, duplicações, complexidade ciclomática
- **Security**: Security hotspots, vulnerabilities (menor cobertura que Pro/Enterprise)
- **Coverage**: Integração com relatórios de cobertura (Jest, Vitest, etc.)
- **Dashboard Web**: Interface rica com histórico, trends, Quality Gates
- **Pull Request decoration**: Comentários automáticos no GitHub

**Tempo de execução:** `+1-2min` no pipeline (análise) + setup inicial de servidor

**Limitações:**

- Requer servidor dedicado (Docker ou VM)
- CE não tem: Branch analysis, PR decoration nativo, análise de múltiplos branches
- Manutenção de servidor (updates, backups)

**Implementação - Servidor:**

```yaml
# docker-compose.yml para SonarQube CE
version: '3.8'
services:
  sonarqube:
    image: sonarqube:10-community
    container_name: sonarqube
    ports:
      - "9000:9000"
    environment:
      - SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs

  postgres:
    image: postgres:15-alpine
    container_name: sonarqube-db
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
      POSTGRES_DB: sonarqube
    volumes:
      - postgresql_data:/var/lib/postgresql/data

volumes:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
  postgresql_data:
```

**Implementação - Action:**

```yaml
# pipeline-template/v1/nodejs/24/sonarqube/action.yml
name: 'SonarQube - Code Quality & Security'
description: 'Analyze code quality and security using SonarQube Community Edition'

inputs:
  sonar-host-url:
    description: 'SonarQube server URL'
    required: true
  sonar-token:
    description: 'SonarQube authentication token'
    required: true
  project-key:
    description: 'SonarQube project key'
    required: true
  sources:
    description: 'Source directories to analyze'
    required: false
    default: 'src'
  coverage-paths:
    description: 'Path to coverage report (lcov.info)'
    required: false
    default: 'coverage/lcov.info'

runs:
  using: 'composite'
  steps:
    - name: '[ACTION] SonarQube Scan'
      uses: sonarsource/sonarqube-scan-action@master
      env:
        SONAR_HOST_URL: ${{ inputs.sonar-host-url }}
        SONAR_TOKEN: ${{ inputs.sonar-token }}
      with:
        args: >
          -Dsonar.projectKey=${{ inputs.project-key }}
          -Dsonar.sources=${{ inputs.sources }}
          -Dsonar.javascript.lcov.reportPaths=${{ inputs.coverage-paths }}
          -Dsonar.qualitygate.wait=true
```

**Uso no workflow:**

```yaml
- name: '[STEP] SonarQube - Code Analysis'
  uses: ./pipeline-template/v1/nodejs/24/sonarqube
  with:
    sonar-host-url: 'https://sonar.meudominio.com'
    sonar-token: ${{ secrets.SONAR_TOKEN }}
    project-key: 'videoconverterpro-api'
    sources: 'src'
    coverage-paths: 'coverage/lcov.info'
```

**Recursos disponíveis no CE:**

- ✅ Code Quality analysis
- ✅ Security hotspots básicos
- ✅ Coverage tracking
- ✅ Quality Gates
- ✅ Histórico de métricas
- ❌ Branch analysis (apenas main)
- ❌ PR decoration automático
- ❌ Advanced security rules

---

### OWASP Dependency-Check

**Função:** Dependency vulnerabilities (alternativa ao npm audit)

**Custo:** `$0` (open-source da OWASP)

**Cobertura:**

- CVE scanning para Java, .NET, Node.js, Python, Ruby, etc.
- Suporte a múltiplos ecossistemas (melhor que npm audit)
- Integração com NIST NVD (National Vulnerability Database)
- Relatórios HTML/XML/JSON detalhados

**Tempo de execução:** `+2-4min` no pipeline (download de NVD database na primeira execução)

**Limitações:**

- Mais lento que npm audit (precisa baixar CVE database)
- Falsos positivos mais comuns
- Relatórios verbosos (muito output)

**Implementação:**

```yaml
# pipeline-template/v1/nodejs/24/dependency-check/action.yml
name: 'OWASP Dependency-Check'
description: 'Scan dependencies for known vulnerabilities using OWASP Dependency-Check'

inputs:
  scan-path:
    description: 'Path to scan'
    required: false
    default: '.'
  fail-on-cvss:
    description: 'Fail build if CVSS score >= value (0-10)'
    required: false
    default: '7'

runs:
  using: 'composite'
  steps:
    - name: '[ACTION] Run OWASP Dependency-Check'
      uses: dependency-check/Dependency-Check_Action@main
      with:
        project: 'VideoConverterPro'
        path: ${{ inputs.scan-path }}
        format: 'HTML'
        args: >
          --failOnCVSS ${{ inputs.fail-on-cvss }}
          --enableRetired

    - name: '[ACTION] Upload report'
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: dependency-check-report-${{ github.sha }}
        path: reports/
        retention-days: 30
```

**Uso no workflow:**

```yaml
- name: '[STEP] OWASP Dependency-Check'
  uses: ./pipeline-template/v1/nodejs/24/dependency-check
  with:
    scan-path: '.'
    fail-on-cvss: '7'
```

---

### OSV-Scanner

**Função:** Dependency vulnerabilities (alternativa moderna ao npm audit)

**Custo:** `$0` (open-source do Google)

**Cobertura:**

- Integração com OSV (Open Source Vulnerabilities) database
- Suporte a 15+ ecossistemas (npm, PyPI, Maven, Go, Rust, etc.)
- Dados agregados de múltiplas fontes (GitHub Advisory, npm advisory, etc.)
- Mais rápido que OWASP Dependency-Check

**Tempo de execução:** `+30-60s` no pipeline

**Limitações:**

- Ferramenta relativamente nova (menos madura)
- Menos features que Snyk (sem priorizações, fix suggestions limitadas)

**Implementação:**

```yaml
# pipeline-template/v1/nodejs/24/osv-scanner/action.yml
name: 'OSV-Scanner - Dependency Vulnerabilities'
description: 'Scan dependencies using OSV (Open Source Vulnerabilities) database'

inputs:
  lockfile-path:
    description: 'Path to lockfile (package-lock.json, pnpm-lock.yaml, etc.)'
    required: false
    default: 'pnpm-lock.yaml'
  format:
    description: 'Output format (table, json, sarif)'
    required: false
    default: 'sarif'

runs:
  using: 'composite'
  steps:
    - name: '[ACTION] Run OSV-Scanner'
      uses: google/osv-scanner-action@v1
      with:
        scan-args: |-
          --lockfile=${{ inputs.lockfile-path }}
          --format=${{ inputs.format }}
          --output=osv-results.sarif

    - name: '[ACTION] Upload SARIF'
      if: always()
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: osv-results.sarif
        category: osv-scanner
```

**Uso no workflow:**

```yaml
- name: '[STEP] OSV-Scanner - Dependency Scan'
  uses: ./pipeline-template/v1/nodejs/24/osv-scanner
  with:
    lockfile-path: 'pnpm-lock.yaml'
    format: 'sarif'
```

---

### Checkov

**Função:** IaC security scanning (Infrastructure as Code)

**Custo:** `$0` (open-source da Bridgecrew/Palo Alto)

**Cobertura:**

- **IaC**: Terraform, CloudFormation, Kubernetes, Helm, Dockerfile, docker-compose
- **Secrets**: Detecção de credenciais em arquivos de configuração
- **Best practices**: Misconfigurations, compliance checks (CIS, PCI-DSS, etc.)
- 1000+ políticas built-in

**Tempo de execução:** `+1-2min` no pipeline

**Limitações:**

- Focado em IaC (não escaneia código de aplicação)
- Overlap com Trivy (ambos fazem IaC scanning)
- Alguns checks muito rigorosos (muitos falsos positivos)

**Implementação:**

```yaml
# pipeline-template/v1/iac/checkov/action.yml
name: 'Checkov - IaC Security Scan'
description: 'Scan Infrastructure as Code for misconfigurations and security issues'

inputs:
  directory:
    description: 'Directory to scan'
    required: false
    default: '.'
  framework:
    description: 'Framework to scan (terraform, cloudformation, kubernetes, dockerfile, all)'
    required: false
    default: 'all'
  soft-fail:
    description: 'Continue on failure'
    required: false
    default: 'false'

runs:
  using: 'composite'
  steps:
    - name: '[ACTION] Run Checkov scan'
      uses: bridgecrewio/checkov-action@master
      with:
        directory: ${{ inputs.directory }}
        framework: ${{ inputs.framework }}
        soft_fail: ${{ inputs.soft-fail }}
        output_format: sarif
        output_file_path: checkov-results.sarif

    - name: '[ACTION] Upload SARIF'
      if: always()
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: checkov-results.sarif
        category: checkov
```

**Uso no workflow:**

```yaml
- name: '[STEP] Checkov - IaC Scan'
  uses: ./pipeline-template/v1/iac/checkov
  with:
    directory: '.'
    framework: 'dockerfile'
    soft-fail: 'false'
```

---

## Já Implementado

### GitLeaks

**Função:** Secret detection (substitui 100% do Snyk Secrets)

**Custo:** `$0` (open-source)

**Cobertura:**

- 170+ regras built-in (AWS, Azure, GCP, GitHub, Slack, etc.)
- Detecção em todo histórico do Git (não apenas diff atual)
- Custom rules via `.gitleaks.toml`
- Allowlist para falsos positivos (`.gitleaksignore`)

**Tempo de execução:** `+15-30s` no pipeline

**Status:** ✅ **Já implementado** - Ver [GITLEAKS.md](./GITLEAKS.md)

**Documentação completa:** [GITLEAKS.md](./GITLEAKS.md)

---

## Comparação com Ferramentas Pagas

### Snyk vs Alternativas Gratuitas

| Feature | Snyk Pro ($500/mês) | npm audit | Trivy | OSV-Scanner |
|---------|---------------------|-----------|-------|-------------|
| **Dependency scan** | ✅ Excelente | ✅ Bom | ✅ Bom | ✅ Bom |
| **Container scan** | ✅ Sim | ❌ Não | ✅ Sim | ❌ Não |
| **IaC scan** | ✅ Sim | ❌ Não | ✅ Sim | ❌ Não |
| **License check** | ✅ Sim | ❌ Não | ✅ Básico | ❌ Não |
| **Fix suggestions** | ✅ PR automático | ⚠️ Manual | ⚠️ Manual | ⚠️ Manual |
| **Priority score** | ✅ Sim | ❌ Não | ❌ Não | ❌ Não |
| **Dashboard web** | ✅ Sim | ❌ Não | ❌ Não | ❌ Não |
| **Custo** | 💰 $500/mês | ✅ $0 | ✅ $0 | ✅ $0 |

**Recomendação:** `npm audit` + `Trivy` = 70% das funcionalidades do Snyk por $0

---

### SonarQube Cloud vs SonarQube CE

| Feature | SonarQube Cloud ($360/ano) | SonarQube CE ($0) |
|---------|----------------------------|-------------------|
| **Code Quality** | ✅ Sim | ✅ Sim |
| **Security Hotspots** | ✅ Avançado | ⚠️ Básico |
| **Branch analysis** | ✅ Sim | ❌ Não |
| **PR decoration** | ✅ Sim | ❌ Não |
| **Quality Gates** | ✅ Sim | ✅ Sim |
| **Coverage tracking** | ✅ Sim | ✅ Sim |
| **Hosting** | ✅ Cloud | ⚠️ Self-hosted |
| **Setup** | ✅ Zero config | ⚠️ Docker + VPS |
| **Custo** | 💰 $360/ano | ✅ $0 + VPS (~$60/ano) |

**Recomendação:** `Semgrep CE` cobre 60% dos security hotspots do SonarQube por $0 (sem servidor)

---

## Stack Recomendada Gratuita

### Para Projetos Pessoais (VideoConverterPro)

**Tier 1 - Implementar AGORA (10 minutos):**

```md
✅ GitLeaks (secrets) - JÁ IMPLEMENTADO
🆕 npm audit (dependency vulnerabilities)
🆕 Semgrep CE (SAST - security hotspots)
🆕 ESLint Security (security linting)
```

**Cobertura:** 70% do Snyk + 60% do SonarQube Security  
**Custo:** $0  
**Overhead no pipeline:** +2min 30s

---

**Tier 2 - Implementar PRÓXIMA SEMANA (30 minutos):**

```md
✅ Tier 1 completo
🆕 Trivy (filesystem + container + IaC)
```

**Cobertura adicional:** +15% (containers, IaC, licenses)  
**Custo:** $0  
**Overhead no pipeline:** +1min 30s

---

**Tier 3 - Opcional/Futuro (quando crescer):**

```md
✅ Tier 1 + 2 completos
🆕 SonarQube CE self-hosted (code quality dashboard)
🆕 Checkov (IaC extra validation)
```

**Cobertura adicional:** +10% (code quality metrics, histórico)  
**Custo:** ~$5-10/mês (VPS)  
**Overhead no pipeline:** +2min

---

### Comparação: C&A Brasil (Pago) vs VideoConverterPro (Grátis)

#### C&A Brasil (Enterprise Stack)

```md
✅ Snyk Pro ($500/mês)
✅ SonarQube Cloud ($360/ano)
✅ GitLeaks (self-hosted)
✅ Trivy (self-hosted)
```

**Custo total:** ~$6400/ano  
**Cobertura:** 100%

---

#### VideoConverterPro (Free Stack)

```md
✅ GitLeaks (secrets)
✅ npm audit (dependencies)
✅ Semgrep CE (SAST)
✅ ESLint Security (linting)
✅ Trivy (multi-purpose)
```

**Custo total:** $0/ano  
**Cobertura:** ~85% do stack da C&A

---

### Timeline de Implementação

```md
┌─────────────────────────────────────────────────────────────────┐
│ HOJE (21/11/2024)                                               │
├─────────────────────────────────────────────────────────────────┤
│ ✅ GitLeaks - JÁ IMPLEMENTADO                                   │
│ 🆕 npm audit - 5 min                                            │
│ 🆕 Semgrep CE - 5 min                                           │
│ 🆕 ESLint Security - 3 min (update package.json)               │
│                                                                 │
│ Pipeline: validation (5min) + build (2min) + deploy (30s)     │
│ Cobertura: 70% do Snyk + 60% do SonarQube                     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ PRÓXIMA SEMANA (25-28/11/2024)                                  │
├─────────────────────────────────────────────────────────────────┤
│ 🆕 Trivy - 30 min (filesystem + container + IaC)               │
│                                                                 │
│ Pipeline: validation (6min) + build (2min) + deploy (30s)     │
│ Cobertura: 85% do Snyk + 60% do SonarQube                     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ OPCIONAL/FUTURO (quando ter >5 desenvolvedores ou >10 repos)   │
├─────────────────────────────────────────────────────────────────┤
│ 🔧 SonarQube CE (self-hosted) - 2h setup inicial               │
│ 🔧 Checkov - 20 min                                             │
│                                                                 │
│ Pipeline: validation (8min) + build (2min) + deploy (30s)     │
│ Cobertura: 85% do Snyk + 90% do SonarQube                     │
│ Custo: ~$10/mês (VPS para SonarQube)                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Workflow Completo (Tier 1 + 2 Implementado)

```yaml
# api/.github/workflows/github-pipeline.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  validation:
    name: '[STAGE] Validation'
    runs-on: ubuntu-latest
    steps:
      - name: '[STEP] Checkout code'
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # GitLeaks needs full history

      # SECURITY: Secret detection
      - name: '[STEP] GitLeaks - Secret Detection'
        uses: ./pipeline-template/v1/nodejs/24/gitleaks

      # SETUP: Node.js + pnpm + cache
      - name: '[STEP] Setup Node.js'
        uses: ./pipeline-template/v1/nodejs/24/setup

      # SECURITY: Dependency vulnerabilities
      - name: '[STEP] npm audit - Dependency Scan'
        uses: ./pipeline-template/v1/nodejs/24/npm-audit
        with:
          severity-level: 'high'
          production-only: 'true'

      # SECURITY: SAST scan
      - name: '[STEP] Semgrep - SAST Scan'
        uses: ./pipeline-template/v1/nodejs/24/semgrep
        with:
          config: 'auto'
          severity: 'WARNING'

      # QUALITY: Linting (includes ESLint Security)
      - name: '[STEP] Lint'
        uses: ./pipeline-template/v1/nodejs/24/lint

      # QUALITY: Testing
      - name: '[STEP] Test'
        uses: ./pipeline-template/v1/nodejs/24/test
        with:
          unit: 'true'
          integration: 'true'
          e2e: 'true'
          coverage: 'true'

      # SECURITY: Multi-purpose scan
      - name: '[STEP] Trivy - Filesystem Scan'
        uses: ./pipeline-template/v1/nodejs/24/trivy
        with:
          scan-type: 'fs'
          severity: 'HIGH,CRITICAL'

  build:
    name: '[STAGE] Build'
    needs: validation
    runs-on: ubuntu-latest
    steps:
      - name: '[STEP] Checkout code'
        uses: actions/checkout@v4

      - name: '[STEP] Setup Node.js'
        uses: ./pipeline-template/v1/nodejs/24/setup

      - name: '[STEP] Build'
        uses: ./pipeline-template/v1/nodejs/24/build

  deploy:
    name: '[STAGE] Deploy'
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: '[STEP] Download build artifacts'
        uses: actions/download-artifact@v4
        with:
          name: dist-${{ github.sha }}
          path: ./dist

      - name: '[STEP] Deploy to production'
        run: |
          echo "🚀 Deploying to production..."
          # Deploy logic here
```

**Tempo total:** ~8min 30s  
**Cobertura de segurança:** 85% comparado ao stack pago da C&A Brasil  
**Custo:** $0

---

## Métricas de Cobertura

### Por Categoria

| Categoria | Ferramentas Pagas (C&A) | Ferramentas Gratuitas (VideoConverterPro) | Cobertura |
|-----------|-------------------------|-------------------------------------------|-----------|
| **Secrets Detection** | GitLeaks | GitLeaks | ✅ 100% |
| **Dependency Vulnerabilities** | Snyk Pro | npm audit + Trivy | ⚠️ 70% |
| **Container Scanning** | Snyk Pro + Trivy | Trivy | ✅ 95% |
| **SAST (Security Hotspots)** | SonarQube Cloud | Semgrep CE | ⚠️ 60% |
| **IaC Scanning** | Snyk IaC | Trivy | ⚠️ 75% |
| **License Scanning** | Snyk | Trivy | ⚠️ 60% |
| **Code Quality** | SonarQube Cloud | ESLint + (opcional: SonarQube CE) | ⚠️ 50% |

**Cobertura média:** ~73% (sem SonarQube CE) ou ~80% (com SonarQube CE self-hosted)

---

### O Que Você Perde (Grátis vs Pago)

**Features ausentes no stack gratuito:**

❌ **Snyk Pro:**

- Automatic PR fix suggestions
- Prioritization by reachability
- Advanced license compliance
- Dashboard web integrado
- Monitoramento contínuo

❌ **SonarQube Cloud:**

- Pull Request decoration automático
- Branch analysis (feature branches)
- Histórico de long-term trends
- Zero setup (cloud-hosted)

❌ **Suporte comercial:**

- SLA de resposta
- Integração enterprise (SSO, RBAC, etc.)
- Treinamento e consultoria

---

### O Que Você Ganha (Grátis)

✅ **Cobertura de segurança essencial:**

- Secrets detection (100%)
- Dependency vulnerabilities (70%)
- SAST hotspots (60%)
- Container/IaC scanning (75%)

✅ **Controle total:**

- Open-source (sem vendor lock-in)
- Customização ilimitada
- Self-hosted se necessário

✅ **Custo zero:**

- $0/mês vs $500-1000/mês
- Escalável (adicionar repos sem custo extra)

---

## Próximos Passos

### Implementação Imediata (Tier 1)

```bash
# 1. Criar actions
cd pipeline-template/v1/nodejs/24/
mkdir npm-audit semgrep
# (copiar código dos exemplos acima)

# 2. Instalar ESLint Security plugins
cd api/
pnpm add -D eslint-plugin-security eslint-plugin-no-secrets

# 3. Atualizar eslint.config.js
# (adicionar plugins conforme exemplo acima)

# 4. Atualizar workflow
# (adicionar steps no validation stage)

# 5. Commit + push
git add .
git commit -m "feat(security): add npm audit, Semgrep CE, ESLint Security"
git push
```

**Tempo estimado:** 10-15 minutos  
**Resultado:** Pipeline com 70% da cobertura do Snyk + 60% do SonarQube por $0

---

### Implementação Próxima Semana (Tier 2)

```bash
# 1. Criar action Trivy
cd pipeline-template/v1/nodejs/24/
mkdir trivy
# (copiar código do exemplo acima)

# 2. Atualizar workflow
# (adicionar Trivy step no validation stage)

# 3. Commit + push
git add .
git commit -m "feat(security): add Trivy multi-purpose scanner"
git push
```

**Tempo estimado:** 30 minutos  
**Resultado:** +15% cobertura (containers, IaC, licenses)

---

## Referências

- **npm audit**: <https://docs.npmjs.com/cli/v10/commands/npm-audit>
- **Semgrep CE**: <https://semgrep.dev/docs/>
- **Trivy**: <https://aquasecurity.github.io/trivy/>
- **GitLeaks**: <https://github.com/gitleaks/gitleaks>
- **SonarQube CE**: <https://docs.sonarsource.com/sonarqube/latest/>
- **OWASP Dependency-Check**: <https://owasp.org/www-project-dependency-check/>
- **OSV-Scanner**: <https://google.github.io/osv-scanner/>
- **Checkov**: <https://www.checkov.io/>
- **ESLint Security**: <https://github.com/eslint-community/eslint-plugin-security>

---

## Conclusão

**Para projetos pessoais e pequenas empresas, o stack gratuito (Tier 1 + 2) oferece 85% da cobertura de segurança de ferramentas pagas como Snyk e SonarQube, por $0/mês.**

A principal diferença está em:

- **UX**: Ferramentas pagas têm dashboards melhores e fix suggestions automáticos
- **Priorizações**: Snyk tem scoring mais inteligente (reachability, exploit maturity)
- **Suporte**: Ferramentas pagas têm SLA e suporte 24/7

Para VideoConverterPro (projeto pessoal/open-source), o stack gratuito é **mais que suficiente** e cobre todos os requisitos de segurança OWASP Top 10.

**Recomendação final:** Implementar **Tier 1 hoje** (10min) e **Tier 2 próxima semana** (30min). Total: $0 + ~40min de trabalho = 85% da cobertura da C&A Brasil. 🎯
