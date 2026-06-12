1. Princípio base

Nenhuma ação do Devin é confiável por padrão.

Toda ação precisa passar por:
1. Identidade do solicitante
2. Organização do solicitante
3. Papel/função do solicitante
4. Finalidade declarada
5. Recurso alvo
6. Política permitida
7. Validação de origem/destino
8. Registro de auditoria

---

Guardrail 1 — Escopo por organização e função
Objetivo

Garantir que um usuário só consiga pedir ao Devin ações compatíveis com sua organização e com sua função.

Exemplo:

```txt
Dev da Organização A:
- Pode criar branch
- Pode abrir PR
- Pode rodar testes
- Pode sugerir alteração de código
- Não pode alterar pipeline de produção
```
Política conceitual

```yml
guardrail:
  name: org_role_scope
  description: "Controla ações permitidas por organização, função e finalidade."

  identity_requirements:
    require_sso: true
    require_mfa: true
    require_user_org_binding: true
    require_role_mapping: true

  allowed_contexts:
    developer:
      purpose:
        - development
        - unit_test
        - code_review
        - documentation
        - dependency_update

      allowed_actions:
        - read_repository
        - create_branch
        - modify_source_code
        - run_unit_tests
        - run_lint
        - open_pull_request
        - comment_on_pull_request

      denied_actions:
        - modify_production_config
        - access_production_secret
        - rotate_secret
        - deploy_to_production
        - disable_security_control
        - change_ci_cd_security_stage
        - modify_branch_protection
        - create_external_network_tunnel
        - execute_unapproved_binary

    appsec:
      purpose:
        - security_review
        - sast_review
        - dependency_review
        - threat_modeling
        - secure_code_fix

      allowed_actions:
        - read_repository
        - inspect_security_findings
        - propose_security_fix
        - run_security_tests
        - open_pull_request
        - generate_security_report

      denied_actions:
        - exfiltrate_source_code
        - bypass_pipeline
        - disable_security_gate
        - approve_own_security_exception
        - deploy_to_production

    admin:
      purpose:
        - platform_administration
        - policy_management

      allowed_actions:
        - manage_roles
        - manage_integrations
        - configure_network_policy
        - configure_artifact_policy

      denied_actions:
        - bypass_audit_log
        - disable_policy_engine_without_breakglass
```
Guardrail 2 — Validação de autorização antes da execução
Objetivo

Antes de executar qualquer ação, o Devin precisa consultar uma política externa, por exemplo:

Devin → Policy Enforcement Point → OPA/Cedar/ABAC/RBAC → Allow/Deny

Decisão esperada

```yml
{
  "subject": {
    "user": "dev01@empresa.com",
    "org": "org-a",
    "role": "developer"
  },
  "action": "modify_source_code",
  "resource": {
    "repository": "org-a/payment-api",
    "branch": "feature/devin-fix"
  },
  "purpose": "development",
  "decision": "allow"
}
```

Exemplo negado:

```yml
{
  "subject": {
    "user": "dev01@empresa.com",
    "org": "org-a",
    "role": "developer"
  },
  "action": "modify_pipeline",
  "resource": {
    "repository": "org-a/payment-api",
    "file": ".github/workflows/security.yml"
  },
  "purpose": "development",
  "decision": "deny",
  "reason": "Developers cannot modify security pipeline files through AI agent."
}
```

Guardrail 3 — Arquivos protegidos

O Devin deve ser impedido de alterar arquivos sensíveis sem aprovação explícita.

```yml
protected_files:
  deny_modification_by_default:
    - ".github/workflows/**"
    - ".gitlab-ci.yml"
    - "Jenkinsfile"
    - "azure-pipelines.yml"
    - "Dockerfile"
    - "docker-compose.yml"
    - "kubernetes/**"
    - "helm/**"
    - "terraform/**"
    - "secrets/**"
    - ".env"
    - ".npmrc"
    - ".pypirc"
    - "package-lock.json"
    - "pnpm-lock.yaml"
    - "yarn.lock"
    - "pom.xml"
    - "build.gradle"
    - "requirements.txt"
    - "pyproject.toml"

  allowed_with_security_review:
    - "dependency manifests"
    - "lockfiles"
    - "container files"
    - "IaC files"
    - "CI/CD definitions"

  required_approvers:
    ci_cd_files:
      - "devsecops"
      - "platform_security"

    dependency_files:
      - "appsec"
      - "tech_lead"

    infrastructure_files:
      - "cloud_security"
      - "platform_owner"
```

Guardrail 4 — Sem chamadas externas diretas
Objetivo

O Devin não deve conseguir fazer chamadas externas livremente.

Isso inclui:

```txt
curl
wget
git clone externo
npm install externo
pip install externo
go get externo
docker pull externo
apt install externo
ssh externo
nc externo
telnet externo
browser externo
API externa
DNS externo não autorizado
```

Política de rede

```yml
network_policy:
  default_egress: deny

  allowed_internal_destinations:
    - github-enterprise.empresa.local
    - gitlab.empresa.local
    - jfrog.empresa.local
    - sonarqube.empresa.local
    - jira.empresa.local
    - confluence.empresa.local
    - vault.empresa.local
    - policy-engine.empresa.local
    - security-scanner.empresa.local
    - logging.empresa.local

  denied_external_destinations:
    - "*"

  denied_protocols:
    - ssh
    - ftp
    - telnet
    - irc
    - tor
    - p2p
    - raw_tcp
    - raw_udp

  allowed_protocols:
    - https_internal_only
```

Na prática, mesmo quando o Devin tiver terminal/browser, a VM ou ambiente dele deve sair apenas por um egress proxy corporativo com allowlist. A documentação de integrações self-hosted/artifacts do Devin cita cenários com artifact repositories atrás de NLB e allowlisting de IP, o que combina com uma abordagem de permitir somente endpoints corporativos aprovados.

Guardrail 5 — Downloads externos somente via broker

Aqui existe um ponto importante: você pediu “sem chamadas externas de nenhuma forma”, mas também pediu que downloads externos passem pelo VirusTotal. Isso cria uma tensão, porque VirusTotal é um serviço externo.

A forma mais segura é:

```txt
Devin não chama VirusTotal diretamente.
Devin não baixa da internet diretamente.

Devin solicita download ao Artifact Broker interno.
Artifact Broker baixa, valida, escaneia e publica no JFrog.
Devin só consome do JFrog interno.
```

Fluxo recomendado

1. Devin solicita dependência ou arquivo
2. Policy Engine valida se o pedido é permitido
3. Artifact Broker verifica origem
4. Broker baixa o artefato em ambiente isolado
5. Broker calcula hash
6. Broker consulta VirusTotal por hash ou submete conforme política interna
7. Broker executa AV/sandbox/SCA
8. Broker valida licença e reputação
9. Broker publica no JFrog
10. Devin instala somente a partir do JFrog

Política

```yml
download_policy:
  default_download: deny

  direct_external_download:
    allowed: false

  allowed_download_path:
    - artifact_broker_internal

  required_checks:
    - source_allowlist
    - hash_calculation
    - virustotal_hash_lookup
    - antivirus_scan
    - malware_sandbox_if_unknown
    - package_signature_validation
    - license_policy_check
    - vulnerability_scan
    - jfrog_artifact_publication

  final_source_for_devin:
    - jfrog.empresa.local

  deny_if:
    - virustotal_malicious_count_greater_than: 0
    - package_has_no_known_publisher: true
    - source_domain_not_approved: true
    - package_is_typosquatting_suspected: true
    - critical_vulnerability_without_exception: true
    - license_not_allowed: true
```
Guardrail 6 — Dependências de desenvolvimento via JFrog

O Devin não deve executar:

npm install lodash
pip install requests
go get github.com/...
mvn dependency:get
docker pull nginx

Diretamente contra a internet.

Ele deve executar somente contra o repositório interno:

npm install --registry=https://jfrog.empresa.local/artifactory/api/npm/npm-virtual
pip install --index-url https://jfrog.empresa.local/artifactory/api/pypi/pypi-virtual/simple
mvn -s settings-corporate.xml
docker pull jfrog.empresa.local/docker/nginx:approved
Regra
dependency_policy:
  package_managers:
    npm:
      allowed_registry:
        - "https://jfrog.empresa.local/artifactory/api/npm/npm-virtual"
      deny_public_registry: true

    pip:
      allowed_registry:
        - "https://jfrog.empresa.local/artifactory/api/pypi/pypi-virtual/simple"
      deny_public_registry: true

    maven:
      allowed_registry:
        - "https://jfrog.empresa.local/artifactory/maven-virtual"
      deny_public_registry: true

    gradle:
      allowed_registry:
        - "https://jfrog.empresa.local/artifactory/gradle-virtual"
      deny_public_registry: true

    docker:
      allowed_registry:
        - "jfrog.empresa.local/docker-approved"
      deny_dockerhub_direct: true
Guardrail 7 — Comandos bloqueados
command_policy:
  default_shell_access: restricted

  denied_commands:
    - "curl *://*"
    - "wget *://*"
    - "nc *"
    - "ncat *"
    - "netcat *"
    - "ssh *"
    - "scp *"
    - "sftp *"
    - "ftp *"
    - "telnet *"
    - "socat *"
    - "python -m http.server"
    - "ruby -run -e httpd"
    - "php -S *"
    - "ngrok *"
    - "cloudflared tunnel *"
    - "tailscale *"
    - "tor *"
    - "proxychains *"

  denied_package_sources:
    - "registry.npmjs.org"
    - "pypi.org"
    - "files.pythonhosted.org"
    - "repo.maven.apache.org"
    - "github.com/*/releases"
    - "raw.githubusercontent.com"
    - "docker.io"
    - "ghcr.io"

  allowed_commands:
    - "npm test"
    - "npm run lint"
    - "npm run build"
    - "pytest"
    - "go test ./..."
    - "mvn test"
    - "gradle test"
    - "git status"
    - "git diff"
    - "git checkout -b feature/*"
Guardrail 8 — Prompt policy para Devin

Esse é o tipo de instrução que eu colocaria como system/developer policy do agente.

Você é um agente de engenharia de software corporativo.

Você só pode executar tarefas compatíveis com:
- A organização do usuário solicitante
- O papel funcional do usuário
- A finalidade declarada da tarefa
- As políticas corporativas de segurança
- Os repositórios autorizados
- Os ambientes autorizados

Você não pode:
- Fazer chamadas externas diretas
- Baixar arquivos diretamente da internet
- Criar túneis de rede
- Executar comandos de rede não autorizados
- Modificar pipelines de segurança
- Remover etapas de SAST, SCA, secret scan, DAST ou compliance
- Alterar branch protection
- Acessar, criar, revelar ou modificar secrets
- Instalar dependências fora do JFrog corporativo
- Executar binários não aprovados
- Fazer deploy em produção
- Aprovar o próprio pull request
- Fazer merge sem aprovação humana

Antes de executar qualquer ação sensível, você deve:
1. Identificar o usuário
2. Identificar a organização
3. Identificar o papel do usuário
4. Identificar o recurso alvo
5. Identificar a finalidade
6. Consultar a política corporativa
7. Explicar a ação pretendida
8. Registrar evidência de execução

Se uma ação for negada, explique o motivo e proponha um caminho seguro.

