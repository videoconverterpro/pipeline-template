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
    - [TruffleHog](#trufflehog)
    - [Bearer CLI](#bearer-cli)
    - [Hadolint](#hadolint)
    - [Syft + Grype](#syft--grype)
    - [Detect-Secrets](#detect-secrets)
    - [CodeQL](#codeql)
    - [Safety](#safety)
    - [Snyk Code (Limited Free)](#snyk-code-limited-free)
    - [OWASP ZAP](#owasp-zap)
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
    - [Tier 1 (Implementado)](#tier-1-implementado)
    - [Tier 2 (Implementado)](#tier-2-implementado)
    - [Tier 3 (Opcional/Futuro)](#tier-3-opcionalfuturo-1)
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

### TruffleHog

**Função:** Secret detection com Machine Learning (alternativa/complemento ao GitLeaks)

**Custo:** `$0` (open-source)

**Cobertura:**

- **700+ detectores** (vs 170+ do GitLeaks)
- **Machine Learning**: Menos falsos positivos que regex puro
- **Verificação de validade**: Testa se secrets estão ativos (faz requests reais)
- Escaneia: Git history, filesystem, APIs (GitHub, GitLab, S3, Docker registries)
- Suporta: AWS, Azure, GCP, GitHub, Slack, Stripe, SendGrid, Mailgun, Twilio, etc.

**Tempo de execução:** `+1-2min` no pipeline (mais lento que GitLeaks devido à verificação)

**Limitações:**

- Mais lento que GitLeaks (verifica validade de cada secret)
- Precisa de acesso à rede (para verificar secrets)
- Alguns serviços podem rate-limit as verificações

**Vantagem sobre GitLeaks:**

- **Verificação ativa**: Informa se secret está válido/ativo
- **ML-based**: Melhor detecção de padrões não óbvios
- **Menos falsos positivos**: ML filtra patterns que parecem secrets mas não são

**Implementação:**

```yaml
# pipeline-template/v1/shared/validations/trufflehog/action.yml
name: 'TruffleHog - ML-based Secret Detection'
description: 'Scan for secrets with machine learning and active verification'

inputs:
  scan-type:
    description: 'Type of scan (git, filesystem, github)'
    required: false
    default: 'git'
  verify:
    description: 'Verify if secrets are active'
    required: false
    default: 'true'
  continue-on-error:
    description: 'Continue pipeline even if secrets found'
    required: false
    default: 'false'

runs:
  using: 'composite'
  steps:
    - name: '[ACTION] Install TruffleHog'
      shell: bash
      run: |
        echo "📦 [TruffleHog] Installing..."
        curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh | sh -s -- -b /usr/local/bin
        trufflehog --version

    - name: '[ACTION] Run TruffleHog scan'
      shell: bash
      run: |
        echo "🔍 [TruffleHog] Scanning for secrets with ML verification..."
        
        VERIFY_FLAG=""
        if [[ "${{ inputs.verify }}" == "true" ]]; then
          VERIFY_FLAG="--verify"
          echo "✅ Secret verification enabled (will test if secrets are active)"
        fi
        
        case "${{ inputs.scan-type }}" in
          git)
            trufflehog git file://. $VERIFY_FLAG --json > trufflehog-results.json
            ;;
          filesystem)
            trufflehog filesystem . $VERIFY_FLAG --json > trufflehog-results.json
            ;;
          github)
            trufflehog github --org=${{ github.repository_owner }} --repo=${{ github.event.repository.name }} $VERIFY_FLAG --json > trufflehog-results.json
            ;;
        esac
        
        FINDINGS=$(jq '. | length' trufflehog-results.json)
        echo "TRUFFLEHOG_FINDINGS=$FINDINGS" >> $GITHUB_ENV
        
        if [[ $FINDINGS -gt 0 ]]; then
          echo "⚠️ Found $FINDINGS secrets!"
          echo "📄 Details in artifact: trufflehog-results.json"
          
          # Show verified secrets (high priority)
          VERIFIED=$(jq '[.[] | select(.Verified == true)] | length' trufflehog-results.json)
          echo "🚨 VERIFIED ACTIVE SECRETS: $VERIFIED"
          
          if [[ $VERIFIED -gt 0 ]]; then
            echo "❌ CRITICAL: Found verified (active) secrets! Rotate immediately!"
            jq -r '.[] | select(.Verified == true) | "\(.DetectorType): \(.Raw) in \(.SourceMetadata.Data.Filesystem.file // .SourceMetadata.Data.Git.file)"' trufflehog-results.json
          fi
          
          if [[ "${{ inputs.continue-on-error }}" != "true" ]]; then
            exit 1
          fi
        else
          echo "✅ No secrets found"
        fi

    - name: '[ACTION] Upload TruffleHog report'
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: trufflehog-report-${{ github.sha }}
        path: trufflehog-results.json
        retention-days: 30
        if-no-files-found: ignore
```

**Uso no workflow:**

```yaml
- name: '[STEP] TruffleHog - ML Secret Detection'
  uses: ./pipeline-template/v1/shared/validations/trufflehog
  with:
    scan-type: 'git'
    verify: 'true'  # Verifica se secrets estão ativos
```

**Combo GitLeaks + TruffleHog (Recomendado):**

```yaml
# Rodando ambos em paralelo para máxima cobertura
- name: '[STEP] GitLeaks - Fast Regex Scan'
  uses: ./pipeline-template/v1/shared/validations/gitleaks
  continue-on-error: true  # Não bloqueia TruffleHog

- name: '[STEP] TruffleHog - ML + Verification'
  uses: ./pipeline-template/v1/shared/validations/trufflehog
  with:
    verify: 'true'
```

**Resultado:** GitLeaks encontra 85% dos secrets rapidamente, TruffleHog valida + encontra os 15% restantes.

---

### Bearer CLI

**Função:** Data Security + Privacy Compliance (GDPR, LGPD, CCPA)

**Custo:** `$0` (open-source)

**Cobertura:**

- **Data leaks**: Detecta PII (emails, CPF, credit cards) em logs, responses, debug code
- **Privacy compliance**: GDPR, LGPD, CCPA, HIPAA violations
- **SAST**: SQL injection, XSS, insecure crypto, hardcoded secrets
- **Data flow**: Rastreia como PII flui pelo código (ex: `user.email` → `console.log()`)
- Suporta: JavaScript, TypeScript, Ruby, Python, PHP, Java, Go

**Tempo de execução:** `+1-2min` no pipeline

**Limitações:**

- Focado em privacy (não substitui Semgrep para SAST geral)
- Alguns falsos positivos em test fixtures
- Relatórios menos detalhados que SonarQube

**Vantagem única:** **Único scanner focado em privacy compliance** - detecta vazamentos de dados pessoais.

**Implementação:**

```yaml
# pipeline-template/v1/shared/validations/bearer/action.yml
name: 'Bearer CLI - Data Security & Privacy'
description: 'Scan for data leaks and privacy compliance issues (GDPR, LGPD)'

inputs:
  severity:
    description: 'Minimum severity (low, medium, high, critical)'
    required: false
    default: 'high'
  report-format:
    description: 'Report format (sarif, json, html)'
    required: false
    default: 'sarif'
  continue-on-error:
    description: 'Continue pipeline even if issues found'
    required: false
    default: 'false'

runs:
  using: 'composite'
  steps:
    - name: '[ACTION] Install Bearer CLI'
      shell: bash
      run: |
        echo "📦 [Bearer] Installing..."
        curl -sfL https://raw.githubusercontent.com/Bearer/bearer/main/contrib/install.sh | sh
        sudo mv ./bearer /usr/local/bin/
        bearer version

    - name: '[ACTION] Run Bearer scan'
      shell: bash
      run: |
        echo "🔍 [Bearer] Scanning for data leaks and privacy issues..."
        
        bearer scan . \
          --format=${{ inputs.report-format }} \
          --output=bearer-results.${{ inputs.report-format }} \
          --severity=${{ inputs.severity }} \
          --quiet
        
        # Parse results
        if [[ "${{ inputs.report-format }}" == "sarif" ]]; then
          FINDINGS=$(jq '[.runs[].results[]] | length' bearer-results.sarif)
        else
          FINDINGS=$(jq '.stats.total' bearer-results.json)
        fi
        
        echo "BEARER_FINDINGS=$FINDINGS" >> $GITHUB_ENV
        
        if [[ $FINDINGS -gt 0 ]]; then
          echo "⚠️ Found $FINDINGS privacy/security issues!"
          echo "📄 Details in artifact: bearer-results.${{ inputs.report-format }}"
          
          # Show summary by category
          echo "📊 Summary:"
          bearer scan . --format=json --quiet | jq -r '.stats'
          
          if [[ "${{ inputs.continue-on-error }}" != "true" ]]; then
            exit 1
          fi
        else
          echo "✅ No privacy/security issues found"
        fi

    - name: '[ACTION] Upload Bearer report'
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: bearer-report-${{ github.sha }}
        path: bearer-results.*
        retention-days: 30
        if-no-files-found: ignore
```

**Uso no workflow:**

```yaml
- name: '[STEP] Bearer - Privacy & Data Security'
  uses: ./pipeline-template/v1/shared/validations/bearer
  with:
    severity: 'high'
    report-format: 'sarif'
```

**Exemplo de detecções:**

```typescript
// ❌ Detectado por Bearer
console.log("User email:", user.email);  // PII leak in logs
res.json({ cpf: person.cpf });            // CPF in API response
logger.debug(creditCard);                 // Credit card in debug logs

// ✅ Bearer sugere:
console.log("User email:", maskEmail(user.email));
res.json({ cpf: maskCPF(person.cpf) });
logger.debug("[REDACTED]");
```

---

### Hadolint

**Função:** Dockerfile linting (best practices + security)

**Custo:** `$0` (open-source)

**Cobertura:**

- **Best practices**: Dockerfile optimization (layer caching, multi-stage builds)
- **Security**: USER root, COPY without .dockerignore, insecure base images
- **Shell validation**: Integra ShellCheck para validar scripts bash no RUN
- Detecta: apt-get without cleanup, missing --no-cache, hardcoded versions, etc.

**Tempo de execução:** `+10-20s` no pipeline

**Limitações:**

- Apenas Dockerfile (não escaneia imagens buildadas)
- Overlap parcial com Trivy IaC (mas Hadolint é mais específico)

**Vantagem:** Especializado em Dockerfile - mais regras que Trivy IaC.

**Implementação:**

```yaml
# pipeline-template/v1/shared/validations/hadolint/action.yml
name: 'Hadolint - Dockerfile Linting'
description: 'Lint Dockerfile for best practices and security issues'

inputs:
  dockerfile-path:
    description: 'Path to Dockerfile'
    required: false
    default: 'Dockerfile'
  severity:
    description: 'Minimum severity (info, warning, error)'
    required: false
    default: 'warning'
  continue-on-error:
    description: 'Continue pipeline even if issues found'
    required: false
    default: 'false'

runs:
  using: 'composite'
  steps:
    - name: '[ACTION] Run Hadolint'
      shell: bash
      run: |
        echo "🔍 [Hadolint] Linting Dockerfile..."
        
        docker run --rm -i \
          -v "$(pwd)":/workspace \
          -w /workspace \
          hadolint/hadolint:latest \
          hadolint \
          --format json \
          --no-fail \
          ${{ inputs.dockerfile-path }} > hadolint-results.json
        
        # Filter by severity
        FINDINGS=$(jq --arg severity "${{ inputs.severity }}" '
          [.[] | select(
            (.level == "error") or
            (.level == "warning" and ($severity == "warning" or $severity == "info")) or
            (.level == "info" and $severity == "info")
          )] | length
        ' hadolint-results.json)
        
        echo "HADOLINT_FINDINGS=$FINDINGS" >> $GITHUB_ENV
        
        if [[ $FINDINGS -gt 0 ]]; then
          echo "⚠️ Found $FINDINGS Dockerfile issues!"
          
          # Show summary
          echo "📊 Issues by severity:"
          jq -r 'group_by(.level) | map({level: .[0].level, count: length}) | .[]' hadolint-results.json
          
          # Show details
          echo "📋 Details:"
          jq -r '.[] | "\(.level | ascii_upcase): \(.message) (\(.code)) at line \(.line)"' hadolint-results.json
          
          if [[ "${{ inputs.continue-on-error }}" != "true" ]]; then
            exit 1
          fi
        else
          echo "✅ No Dockerfile issues found"
        fi

    - name: '[ACTION] Upload Hadolint report'
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: hadolint-report-${{ github.sha }}
        path: hadolint-results.json
        retention-days: 30
        if-no-files-found: ignore
```

**Uso no workflow:**

```yaml
- name: '[STEP] Hadolint - Dockerfile Lint'
  uses: ./pipeline-template/v1/shared/validations/hadolint
  with:
    dockerfile-path: 'Dockerfile'
    severity: 'warning'
```

**Exemplo de detecções:**

```dockerfile
# ❌ Detectado por Hadolint
FROM node:latest                    # DL3007: Use versão específica
RUN apt-get update && apt-get install curl  # DL3008: Pin versions
COPY . .                            # DL3045: Use .dockerignore
USER root                           # DL3002: Não use root

# ✅ Hadolint recomenda:
FROM node:20-alpine
RUN apt-get update && apt-get install -y curl=7.68.0-1 && rm -rf /var/lib/apt/lists/*
COPY --chown=node:node . .
USER node
```

---

### Syft + Grype

**Função:** SBOM generation + vulnerability scanning (alternativa ao Trivy)

**Custo:** `$0` (open-source da Anchore)

**Cobertura:**

- **Syft**: Gera SBOM (Software Bill of Materials) em formato SPDX, CycloneDX, Syft JSON
- **Grype**: Escaneia vulnerabilidades usando SBOM (vs databases: NVD, GitHub, etc.)
- Suporta: Container images, filesystems, archives, language packages
- Detecta: OS packages, language libraries (npm, pip, gem, go.mod, etc.)

**Tempo de execução:** `+1-2min` no pipeline

**Limitações:**

- Dois comandos separados (Syft → Grype)
- Overlap com Trivy (ambos fazem vulnerability scan)
- SBOM generation adiciona overhead

**Vantagem sobre Trivy:**

- **SBOM completo**: Útil para compliance/auditoria (ISO 27001, SOC 2)
- **Formatos padrão**: SPDX 2.3, CycloneDX 1.4 (interoperável com outras ferramentas)
- **Database próprio**: Não depende de NVD (menos downtime)

**Implementação:**

```yaml
# pipeline-template/v1/shared/validations/syft-grype/action.yml
name: 'Syft + Grype - SBOM & Vulnerability Scan'
description: 'Generate SBOM and scan vulnerabilities using Anchore tools'

inputs:
  scan-target:
    description: 'Target to scan (dir:., image:myapp:latest)'
    required: false
    default: 'dir:.'
  sbom-format:
    description: 'SBOM format (spdx-json, cyclonedx-json, syft-json)'
    required: false
    default: 'spdx-json'
  severity:
    description: 'Minimum severity (negligible, low, medium, high, critical)'
    required: false
    default: 'high'
  continue-on-error:
    description: 'Continue pipeline even if vulnerabilities found'
    required: false
    default: 'false'

runs:
  using: 'composite'
  steps:
    - name: '[ACTION] Install Syft'
      shell: bash
      run: |
        echo "📦 [Syft] Installing..."
        curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
        syft version

    - name: '[ACTION] Install Grype'
      shell: bash
      run: |
        echo "📦 [Grype] Installing..."
        curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin
        grype version

    - name: '[ACTION] Generate SBOM with Syft'
      shell: bash
      run: |
        echo "📄 [Syft] Generating SBOM..."
        syft ${{ inputs.scan-target }} \
          -o ${{ inputs.sbom-format }} \
          --file sbom.${{ inputs.sbom-format }}
        
        PACKAGES=$(jq '[.packages // .components] | length' sbom.${{ inputs.sbom-format }})
        echo "✅ SBOM generated: $PACKAGES packages found"

    - name: '[ACTION] Scan vulnerabilities with Grype'
      shell: bash
      run: |
        echo "🔍 [Grype] Scanning vulnerabilities from SBOM..."
        
        grype sbom:sbom.${{ inputs.sbom-format }} \
          --fail-on ${{ inputs.severity }} \
          -o json \
          --file grype-results.json
        
        VULNS=$(jq '[.matches[]] | length' grype-results.json)
        echo "GRYPE_VULNS=$VULNS" >> $GITHUB_ENV
        
        if [[ $VULNS -gt 0 ]]; then
          echo "⚠️ Found $VULNS vulnerabilities!"
          
          # Summary by severity
          echo "📊 Summary:"
          jq -r '[.matches[].vulnerability.severity] | group_by(.) | map({severity: .[0], count: length}) | .[]' grype-results.json
          
          if [[ "${{ inputs.continue-on-error }}" != "true" ]]; then
            exit 1
          fi
        else
          echo "✅ No vulnerabilities found"
        fi

    - name: '[ACTION] Upload SBOM'
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: sbom-${{ github.sha }}
        path: sbom.${{ inputs.sbom-format }}
        retention-days: 90  # SBOM deve ser mantido por mais tempo
        if-no-files-found: ignore

    - name: '[ACTION] Upload Grype report'
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: grype-report-${{ github.sha }}
        path: grype-results.json
        retention-days: 30
        if-no-files-found: ignore
```

**Uso no workflow:**

```yaml
# Filesystem scan
- name: '[STEP] Syft + Grype - SBOM & Vuln Scan'
  uses: ./pipeline-template/v1/shared/validations/syft-grype
  with:
    scan-target: 'dir:.'
    sbom-format: 'spdx-json'
    severity: 'high'

# Container scan
- name: '[STEP] Build Docker image'
  run: docker build -t myapp:${{ github.sha }} .

- name: '[STEP] Syft + Grype - Container Scan'
  uses: ./pipeline-template/v1/shared/validations/syft-grype
  with:
    scan-target: 'image:myapp:${{ github.sha }}'
    sbom-format: 'cyclonedx-json'
```

---

### Detect-Secrets

**Função:** Secret detection com baseline approach (alternativa leve ao GitLeaks)

**Custo:** `$0` (open-source da Yelp)

**Cobertura:**

- **Baseline tracking**: Armazena secrets conhecidos em `.secrets.baseline`
- **Incremental scan**: Apenas novos commits (não re-escaneia histórico)
- **Plugins customizáveis**: Regex, entropy, keywords, AWS, Azure, etc.
- Mais leve e rápido que GitLeaks (ideal para CI/CD)

**Tempo de execução:** `+10-15s` no pipeline (vs +30s do GitLeaks)

**Limitações:**

- Menos regras padrão que GitLeaks (170+) ou TruffleHog (700+)
- Precisa manter `.secrets.baseline` atualizado
- Sem verificação de validade (diferente do TruffleHog)

**Vantagem:** Baseline approach evita re-escanear histórico antigo - ideal para repos grandes.

**Implementação:**

```yaml
# pipeline-template/v1/shared/validations/detect-secrets/action.yml
name: 'Detect-Secrets - Baseline Secret Detection'
description: 'Scan for secrets using baseline approach (incremental)'

inputs:
  baseline-file:
    description: 'Path to baseline file'
    required: false
    default: '.secrets.baseline'
  update-baseline:
    description: 'Update baseline with new findings'
    required: false
    default: 'false'
  continue-on-error:
    description: 'Continue pipeline even if new secrets found'
    required: false
    default: 'false'

runs:
  using: 'composite'
  steps:
    - name: '[ACTION] Install detect-secrets'
      shell: bash
      run: |
        echo "📦 [detect-secrets] Installing..."
        pip install detect-secrets
        detect-secrets --version

    - name: '[ACTION] Create baseline if not exists'
      shell: bash
      run: |
        if [[ ! -f "${{ inputs.baseline-file }}" ]]; then
          echo "📄 Creating new baseline..."
          detect-secrets scan > ${{ inputs.baseline-file }}
        fi

    - name: '[ACTION] Scan for new secrets'
      shell: bash
      run: |
        echo "🔍 [detect-secrets] Scanning for NEW secrets..."
        
        # Scan and compare with baseline
        detect-secrets scan > .secrets.new
        
        # Check for new secrets
        detect-secrets audit --diff ${{ inputs.baseline-file }} .secrets.new > detect-secrets-diff.json
        
        NEW_SECRETS=$(jq '.results | length' detect-secrets-diff.json)
        echo "DETECT_SECRETS_NEW=$NEW_SECRETS" >> $GITHUB_ENV
        
        if [[ $NEW_SECRETS -gt 0 ]]; then
          echo "⚠️ Found $NEW_SECRETS NEW secrets!"
          echo "📋 New secrets:"
          jq -r '.results[] | "\(.filename):\(.line_number) - \(.type)"' detect-secrets-diff.json
          
          if [[ "${{ inputs.update-baseline }}" == "true" ]]; then
            echo "✏️ Updating baseline..."
            mv .secrets.new ${{ inputs.baseline-file }}
            echo "⚠️ Baseline updated - commit the changes!"
          fi
          
          if [[ "${{ inputs.continue-on-error }}" != "true" ]]; then
            exit 1
          fi
        else
          echo "✅ No new secrets found"
        fi

    - name: '[ACTION] Upload report'
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: detect-secrets-report-${{ github.sha }}
        path: |
          detect-secrets-diff.json
          .secrets.baseline
        retention-days: 30
        if-no-files-found: ignore
```

**Uso no workflow:**

```yaml
- name: '[STEP] Detect-Secrets - Incremental Scan'
  uses: ./pipeline-template/v1/shared/validations/detect-secrets
  with:
    baseline-file: '.secrets.baseline'
    update-baseline: 'false'  # true = atualiza baseline automaticamente
```

**Workflow recomendado:**

1. **Primeira execução**: `update-baseline: 'true'` (cria baseline inicial)
2. **CI/CD normal**: `update-baseline: 'false'` (detecta novos secrets)
3. **Após rotacionar secret**: `update-baseline: 'true'` + commit baseline

---

### CodeQL

**Função:** SAST avançado (análise de fluxo de dados completa)

**Custo:** `$0` (repos públicos) | **$49/mês/dev** (repos privados com GitHub Advanced Security)

**Cobertura:**

- **Dataflow analysis**: Rastreia dados do input até sink (ex: `req.body` → SQL query)
- **2000+ queries** para JavaScript/TypeScript
- **Custom queries**: QL language (Datalog-like) para criar regras próprias
- **Integração nativa**: GitHub Security tab, PR annotations
- Suporta: JavaScript, TypeScript, Python, Java, C/C++, C#, Go, Ruby

**Tempo de execução:** `+3-5min` no pipeline

**Limitações:**

- **Requer GitHub Advanced Security** para repos privados ($49/mês/dev)
- Mais lento que Semgrep (análise mais profunda)
- Setup mais complexo (precisa criar CodeQL database)

**Vantagem sobre Semgrep:**

- **Análise de fluxo completa**: Detecta vulnerabilidades complexas (Semgrep é pattern-matching)
- **Menos falsos positivos**: Entende contexto completo do código
- **GitHub native**: Integra perfeitamente com GitHub Security

**Implementação:**

```yaml
# pipeline-template/v1/shared/validations/codeql/action.yml
name: 'CodeQL - Advanced SAST'
description: 'Static analysis with dataflow tracking using CodeQL'

inputs:
  languages:
    description: 'Languages to analyze (comma-separated: javascript, python, java, etc)'
    required: false
    default: 'javascript'
  queries:
    description: 'Query suite (security-extended, security-and-quality)'
    required: false
    default: 'security-extended'

runs:
  using: 'composite'
  steps:
    - name: '[ACTION] Initialize CodeQL'
      uses: github/codeql-action/init@v3
      with:
        languages: ${{ inputs.languages }}
        queries: ${{ inputs.queries }}

    - name: '[ACTION] Autobuild'
      uses: github/codeql-action/autobuild@v3

    - name: '[ACTION] Perform CodeQL Analysis'
      uses: github/codeql-action/analyze@v3
      with:
        category: codeql-analysis
```

**Uso no workflow:**

```yaml
- name: '[STEP] CodeQL - Advanced SAST'
  uses: ./pipeline-template/v1/shared/validations/codeql
  with:
    languages: 'javascript,typescript'
    queries: 'security-extended'
```

**⚠️ IMPORTANTE:**

- **Repos públicos**: Grátis, funciona sem configuração adicional
- **Repos privados**: Requer GitHub Advanced Security (pago)
- **Alternativa gratuita para repos privados**: Use Semgrep CE (70% da cobertura)

---

### Safety

**Função:** Dependency vulnerabilities para Python (equivalente ao npm audit)

**Custo:** `$0` (database público da PyUp.io)

**Cobertura:**

- Escaneia: `requirements.txt`, `Pipfile`, `Pipfile.lock`, `poetry.lock`, `pyproject.toml`
- Database de CVEs Python (mantido pela PyUp.io)
- Classificação por severidade
- Suggestions de fix (upgrade version)

**Tempo de execução:** `+20-30s` no pipeline

**Limitações:**

- Apenas Python (não escaneia JavaScript, Go, etc.)
- Database menor que NVD (mas focado em Python)
- Alguns CVEs demoram para serem adicionados

**Uso:** Se você tiver scripts Python no projeto (ex: `prisma/seed.py`, `scripts/migrate.py`).

**Implementação:**

```yaml
# pipeline-template/v1/python/shared/validations/safety/action.yml
name: 'Safety - Python Dependency Scan'
description: 'Scan Python dependencies for known vulnerabilities'

inputs:
  requirements-file:
    description: 'Path to requirements file'
    required: false
    default: 'requirements.txt'
  continue-on-error:
    description: 'Continue pipeline even if vulnerabilities found'
    required: false
    default: 'false'

runs:
  using: 'composite'
  steps:
    - name: '[ACTION] Install Safety'
      shell: bash
      run: |
        echo "📦 [Safety] Installing..."
        pip install safety
        safety --version

    - name: '[ACTION] Run Safety check'
      shell: bash
      run: |
        echo "🔍 [Safety] Scanning Python dependencies..."
        
        safety check \
          --file ${{ inputs.requirements-file }} \
          --json \
          --output safety-results.json \
          --continue-on-error
        
        VULNS=$(jq '[.vulnerabilities[]] | length' safety-results.json)
        echo "SAFETY_VULNS=$VULNS" >> $GITHUB_ENV
        
        if [[ $VULNS -gt 0 ]]; then
          echo "⚠️ Found $VULNS vulnerabilities in Python dependencies!"
          
          # Show affected packages
          echo "📦 Affected packages:"
          jq -r '.vulnerabilities[] | "\(.package): \(.installed_version) → \(.vulnerable_spec) (\(.vulnerability_id))"' safety-results.json
          
          if [[ "${{ inputs.continue-on-error }}" != "true" ]]; then
            exit 1
          fi
        else
          echo "✅ No vulnerabilities found in Python dependencies"
        fi

    - name: '[ACTION] Upload Safety report'
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: safety-report-${{ github.sha }}
        path: safety-results.json
        retention-days: 30
        if-no-files-found: ignore
```

**Uso no workflow:**

```yaml
- name: '[STEP] Safety - Python Dependency Scan'
  uses: ./pipeline-template/v1/python/shared/validations/safety
  with:
    requirements-file: 'requirements.txt'
```

---

### Snyk Code (Limited Free)

**Função:** SAST cloud-based com AI (alternativa ao Semgrep)

**Custo:** `$0` (**500 scans/mês** em repos públicos) | $25/mês (unlimited)

**Cobertura:**

- **DeepCode AI**: Machine learning para detectar vulnerabilidades
- **Fix suggestions automáticas**: PR comments com código corrigido
- **Faster than local SAST**: Cloud-based (não roda no runner)
- Suporta: JavaScript, TypeScript, Python, Java, C#, Go, PHP, Ruby

**Tempo de execução:** `+30-60s` no pipeline (cloud-based)

**Limitações:**

- **500 scans/mês no free tier** (ideal para projetos pessoais, não empresas)
- Precisa de conta Snyk (signup gratuito)
- Envia código para cloud Snyk (privacy concern?)

**Vantagem sobre Semgrep:**

- **AI-powered**: Detecta patterns que Semgrep (regex) não pega
- **Fix suggestions**: PR comments automáticos com código corrigido
- **Mais rápido**: Cloud-based (vs Semgrep local)

**Implementação:**

```yaml
# pipeline-template/v1/shared/validations/snyk-code/action.yml
name: 'Snyk Code - AI-powered SAST'
description: 'SAST with AI using Snyk Code (500 free scans/month)'

inputs:
  snyk-token:
    description: 'Snyk API token (get from https://snyk.io/account)'
    required: true
  severity:
    description: 'Minimum severity (low, medium, high)'
    required: false
    default: 'high'
  continue-on-error:
    description: 'Continue pipeline even if issues found'
    required: false
    default: 'false'

runs:
  using: 'composite'
  steps:
    - name: '[ACTION] Run Snyk Code'
      uses: snyk/actions/node@master
      env:
        SNYK_TOKEN: ${{ inputs.snyk-token }}
      with:
        command: code test
        args: --severity-threshold=${{ inputs.severity }} --json-file-output=snyk-code-results.json

    - name: '[ACTION] Parse results'
      shell: bash
      run: |
        ISSUES=$(jq '[.runs[].results[]] | length' snyk-code-results.json)
        echo "SNYK_CODE_ISSUES=$ISSUES" >> $GITHUB_ENV
        
        if [[ $ISSUES -gt 0 ]]; then
          echo "⚠️ Found $ISSUES security issues!"
          
          # Show summary
          echo "📊 Summary:"
          jq -r '.runs[].results[] | "\(.ruleId): \(.message.text) in \(.locations[0].physicalLocation.artifactLocation.uri)"' snyk-code-results.json | head -20
          
          if [[ "${{ inputs.continue-on-error }}" != "true" ]]; then
            exit 1
          fi
        else
          echo "✅ No security issues found"
        fi

    - name: '[ACTION] Upload Snyk Code report'
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: snyk-code-report-${{ github.sha }}
        path: snyk-code-results.json
        retention-days: 30
        if-no-files-found: ignore
```

**Uso no workflow:**

```yaml
- name: '[STEP] Snyk Code - AI SAST'
  uses: ./pipeline-template/v1/shared/validations/snyk-code
  with:
    snyk-token: ${{ secrets.SNYK_TOKEN }}  # Criar secret no GitHub
    severity: 'high'
```

**⚠️ Setup necessário:**

1. Criar conta gratuita: <https://snyk.io/signup>
2. Obter API token: <https://snyk.io/account>
3. Adicionar `SNYK_TOKEN` nos secrets do GitHub

---

### OWASP ZAP

**Função:** DAST (Dynamic Application Security Testing)

**Custo:** `$0` (open-source)

**Cobertura:**

- **Runtime testing**: Escaneia app **em execução** (complementa SAST)
- **OWASP Top 10**: SQL injection, XSS, CSRF, broken auth, etc.
- **Spider + Active Scan**: Crawls app e testa todas as rotas
- **API scanning**: Suporta REST, GraphQL, SOAP

**Tempo de execução:** `+5-10min` no pipeline (depende do tamanho da app)

**Limitações:**

- **Precisa de app rodando**: Não funciona com código estático
- **Mais lento**: Faz requests HTTP reais (vs SAST que analisa código)
- **Staging only**: Não rode em produção (pode causar danos)

**Vantagem:** **Único DAST gratuito** - detecta vulnerabilidades que SAST não consegue (runtime behavior).

**Implementação:**

```yaml
# pipeline-template/v1/shared/validations/owasp-zap/action.yml
name: 'OWASP ZAP - DAST'
description: 'Dynamic security testing (scans running application)'

inputs:
  target-url:
    description: 'URL of the running application to scan'
    required: true
  scan-type:
    description: 'Type of scan (baseline, full, api)'
    required: false
    default: 'baseline'
  continue-on-error:
    description: 'Continue pipeline even if issues found'
    required: false
    default: 'false'

runs:
  using: 'composite'
  steps:
    - name: '[ACTION] Run OWASP ZAP scan'
      shell: bash
      run: |
        echo "🔍 [OWASP ZAP] Scanning running application..."
        
        case "${{ inputs.scan-type }}" in
          baseline)
            docker run --rm \
              -v $(pwd):/zap/wrk/:rw \
              ghcr.io/zaproxy/zaproxy:stable \
              zap-baseline.py \
              -t ${{ inputs.target-url }} \
              -J zap-report.json \
              -r zap-report.html
            ;;
          full)
            docker run --rm \
              -v $(pwd):/zap/wrk/:rw \
              ghcr.io/zaproxy/zaproxy:stable \
              zap-full-scan.py \
              -t ${{ inputs.target-url }} \
              -J zap-report.json \
              -r zap-report.html
            ;;
          api)
            docker run --rm \
              -v $(pwd):/zap/wrk/:rw \
              ghcr.io/zaproxy/zaproxy:stable \
              zap-api-scan.py \
              -t ${{ inputs.target-url }} \
              -f openapi \
              -J zap-report.json \
              -r zap-report.html
            ;;
        esac
        
        # Parse results
        ALERTS=$(jq '[.site[].alerts[]] | length' zap-report.json)
        echo "ZAP_ALERTS=$ALERTS" >> $GITHUB_ENV
        
        if [[ $ALERTS -gt 0 ]]; then
          echo "⚠️ Found $ALERTS security issues in running app!"
          
          # Show high/medium issues
          echo "🚨 High/Medium issues:"
          jq -r '.site[].alerts[] | select(.riskcode | tonumber >= 2) | "\(.risk): \(.alert) in \(.url)"' zap-report.json
          
          if [[ "${{ inputs.continue-on-error }}" != "true" ]]; then
            exit 1
          fi
        else
          echo "✅ No security issues found"
        fi

    - name: '[ACTION] Upload ZAP reports'
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: zap-report-${{ github.sha }}
        path: |
          zap-report.json
          zap-report.html
        retention-days: 30
        if-no-files-found: ignore
```

**Uso no workflow:**

```yaml
# Precisa rodar app primeiro
- name: '[STEP] Start app'
  run: |
    npm run start &
    sleep 10  # Aguarda app inicializar

- name: '[STEP] OWASP ZAP - DAST'
  uses: ./pipeline-template/v1/shared/validations/owasp-zap
  with:
    target-url: 'http://localhost:3000'
    scan-type: 'baseline'

- name: '[STEP] Stop app'
  if: always()
  run: pkill -f "npm run start"
```

**Recomendação:** Use apenas em **staging environment**, não em produção!

---

## Já Implementado

### GitLeaks

**Função:** Secret detection (substitui 100% do Snyk Secrets)

**Custo:** `$0` (open-source)

**Status:** ✅ **Já implementado**

**Documentação completa:** Ver [GITLEAKS.md](./GITLEAKS.md) para:

- 170+ regras built-in (AWS, Azure, GCP, GitHub, Slack, etc.)
- Configuração detalhada (inputs, .gitleaks.toml, .gitleaksignore)
- Exemplos de uso e integração
- Troubleshooting e best practices
- Como rotacionar secrets detectados

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

### Tier 1 (Implementado)

- **npm audit**: <https://docs.npmjs.com/cli/v10/commands/npm-audit>
- **Semgrep CE**: <https://semgrep.dev/docs/>
- **ESLint Security**: <https://github.com/eslint-community/eslint-plugin-security>
- **GitLeaks**: <https://github.com/gitleaks/gitleaks> | [📖 Docs](./GITLEAKS.md)

### Tier 2 (Implementado)

- **Trivy**: <https://aquasecurity.github.io/trivy/>

### Tier 3 (Opcional/Futuro)

- **SonarQube CE**: <https://docs.sonarsource.com/sonarqube/latest/>
- **OWASP Dependency-Check**: <https://owasp.org/www-project-dependency-check/>
- **OSV-Scanner**: <https://google.github.io/osv-scanner/>
- **Checkov**: <https://www.checkov.io/>
- **TruffleHog**: <https://github.com/trufflesecurity/trufflehog>
- **Bearer CLI**: <https://docs.bearer.com/>
- **Hadolint**: <https://github.com/hadolint/hadolint>
- **Syft**: <https://github.com/anchore/syft>
- **Grype**: <https://github.com/anchore/grype>
- **Detect-Secrets**: <https://github.com/Yelp/detect-secrets>
- **CodeQL**: <https://codeql.github.com/>
- **Safety**: <https://pyup.io/safety/>
- **Snyk Code**: <https://snyk.io/product/snyk-code/>
- **OWASP ZAP**: <https://www.zaproxy.org/>

---

## Conclusão

**Para projetos pessoais e pequenas empresas, o stack gratuito (Tier 1 + 2) oferece 85% da cobertura de segurança de ferramentas pagas como Snyk e SonarQube, por $0/mês.**

A principal diferença está em:

- **UX**: Ferramentas pagas têm dashboards melhores e fix suggestions automáticos
- **Priorizações**: Snyk tem scoring mais inteligente (reachability, exploit maturity)
- **Suporte**: Ferramentas pagas têm SLA e suporte 24/7

Para VideoConverterPro (projeto pessoal/open-source), o stack gratuito é **mais que suficiente** e cobre todos os requisitos de segurança OWASP Top 10.

**Recomendação final:** Implementar **Tier 1 hoje** (10min) e **Tier 2 próxima semana** (30min). Total: $0 + ~40min de trabalho = 85% da cobertura da C&A Brasil. 🎯
