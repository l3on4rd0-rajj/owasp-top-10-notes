---
tags: [security, owasp, api8, security-misconfiguration, hardening, cors, headers, devsecops, pentest]
aliases:
  - API8 Security Misconfiguration
  - Security Misconfiguration
  - Configuração Incorreta de Segurança
  - Misconfiguration
---

# 🧯 API8:2023 — Security Misconfiguration

**Resumo:**  
**Security Misconfiguration** ocorre quando uma API, serviço, infraestrutura ou componente de suporte é implantado com configurações inseguras, excessivamente permissivas, padrões de fábrica ou recursos de debug/diagnóstico expostos.

Em APIs, esse risco costuma aparecer em CORS mal configurado, mensagens de erro verbosas, documentação exposta, endpoints de debug, headers ausentes, permissões cloud excessivas e ambientes de desenvolvimento acessíveis publicamente.

---

## 🔗 Associações no Obsidian

- Hub: [[API Security]]
- Infra e hardening: [[Containers e Orquestração]], [[Docker — Guia Rápido (Instalação, Comandos e Dockerfile)]], [[docker compose]], [[minikube]]
- DevSecOps e revisão: [[GHAS]], [[github]], [[improper-inventory-management|API9:2023 — Improper Inventory Management]], [[Vazamentos e Tokens de API]]
- Testes de API: [[Guia de consulta rápida — Avaliação REST]], [[Pentest]], [[Vulnerabilidades]]
- Resposta e investigação: [[Segurança de Aplicativos - Resposta a Incidentes]]

---

## 🧠 Descrição da Vulnerabilidade

Configurações incorretas podem surgir em várias camadas:

- Código da API.
- Framework web.
- Servidor HTTP ou proxy reverso.
- Gateway/API Gateway.
- Containers e imagens.
- Kubernetes.
- Cloud e IAM.
- Banco de dados.
- Pipelines e repositórios.
- Observabilidade, logs e painéis administrativos.

O problema central é que a API fica mais exposta do que deveria ou revela informações úteis para exploração.

---

## ⚠️ A API é Vulnerável?

A API pode estar vulnerável se:

- Retorna stack traces ou mensagens de erro detalhadas em produção.
- Possui CORS aberto com credenciais habilitadas.
- Expõe Swagger/OpenAPI sem autenticação em produção.
- Mantém endpoints de debug, health detalhado ou actuator expostos.
- Usa configurações padrão, senhas padrão ou secrets fracos.
- Permite métodos HTTP desnecessários.
- Não define headers de segurança.
- Expõe buckets, volumes, backups, arquivos `.env` ou logs.
- Executa containers como `root` sem necessidade.
- Possui permissões cloud amplas demais.
- Usa ambientes de desenvolvimento/staging com dados reais e acesso público.

---

## 💣 Cenários de Ataque

### 🧨 Cenário #1 — CORS refletindo origens com credenciais

Configuração insegura:

```http
Origin: https://evil.example

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://evil.example
Access-Control-Allow-Credentials: true
```

➡️ **Problema:** a API reflete qualquer origem recebida no header `Origin` e ainda permite credenciais.

➡️ **Resultado:** um site malicioso pode tentar interagir com a API no contexto autenticado do navegador da vítima.

### 🧨 Cenário #2 — Swagger exposto em produção

Endpoint público:

```http
GET /swagger-ui/index.html
GET /v3/api-docs
```

➡️ **Problema:** a documentação revela rotas, schemas, parâmetros e endpoints administrativos.

➡️ **Resultado:** o atacante ganha um mapa detalhado da superfície da API.

### 🧨 Cenário #3 — Stack trace em erro de produção

Resposta de erro:

```http
HTTP/1.1 500 Internal Server Error
Content-Type: application/json

{
  "error": "NullPointerException",
  "class": "com.company.auth.TokenService",
  "line": 87,
  "db": "jdbc:postgresql://internal-db:5432/app"
}
```

➡️ **Problema:** a API revela detalhes internos de código e infraestrutura.

➡️ **Resultado:** facilita reconhecimento, exploração e movimentação lateral.

### 🧨 Cenário #4 — Actuator/debug exposto

Endpoint exposto:

```http
GET /actuator/env
GET /actuator/heapdump
GET /debug/vars
```

➡️ **Problema:** endpoints administrativos ou de diagnóstico estão acessíveis sem proteção.

➡️ **Resultado:** exposição de variáveis, configurações, dados sensíveis ou estado interno.

---

## 💻 Exemplos de Código — Vulnerável e Mitigado

### ❌ Node.js (Express) — Exemplo Vulnerável

```javascript
import cors from 'cors';
import express from 'express';

const app = express();

app.use(cors({
  origin: true,
  credentials: true,
}));

app.use((err, req, res, next) => {
  res.status(500).json({
    message: err.message,
    stack: err.stack,
  });
});
```

**Problemas:**

- CORS reflete qualquer origem recebida.
- Credenciais habilitadas junto com origem não confiável.
- Stack trace exposto ao cliente.

### ✅ Node.js — Mitigação Recomendada

```javascript
import cors from 'cors';
import express from 'express';
import helmet from 'helmet';

const app = express();

const allowedOrigins = new Set([
  'https://app.example.com',
  'https://admin.example.com',
]);

app.use(helmet());

app.use(cors({
  origin(origin, callback) {
    if (!origin || allowedOrigins.has(origin)) {
      return callback(null, true);
    }

    return callback(new Error('Origem não permitida'));
  },
  credentials: true,
}));

app.use((err, req, res, next) => {
  req.log?.error({ err }, 'Erro interno');

  res.status(500).json({
    error: 'Erro interno',
  });
});
```

**Mitigação aplicada:**

- Allowlist de origens.
- Headers de segurança com `helmet`.
- Erros genéricos para o cliente.
- Detalhes técnicos apenas em logs internos.

### ❌ Java (Spring Boot) — Exemplo Vulnerável

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"

server:
  error:
    include-stacktrace: always
    include-message: always
```

**Problemas:**

- Todos os endpoints actuator expostos.
- Stack traces e mensagens internas sempre retornados.

### ✅ Java (Spring Boot) — Mitigação Recomendada

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "health,info"
  endpoint:
    health:
      show-details: never

server:
  error:
    include-stacktrace: never
    include-message: never
```

```java
@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/actuator/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .build();
    }
}
```

**Mitigação aplicada:**

- Exposição mínima de actuator.
- Stack traces ocultos.
- Endpoints administrativos protegidos por role.

---

## 🛡️ Como Prevenir

- [ ] Aplicar hardening por ambiente antes de publicar a API.
- [ ] Remover configurações padrão e recursos de debug em produção.
- [ ] Restringir CORS com allowlist explícita.
- [ ] Exibir mensagens de erro genéricas ao usuário.
- [ ] Proteger Swagger/OpenAPI, actuator, métricas e painéis administrativos.
- [ ] Desabilitar métodos HTTP desnecessários.
- [ ] Configurar headers de segurança.
- [ ] Rodar containers com usuário não-root quando possível.
- [ ] Evitar secrets em variáveis, imagens, logs e repositórios sem proteção.
- [ ] Aplicar menor privilégio em IAM, banco, tokens e service accounts.
- [ ] Validar configurações com scanners e revisões automatizadas.

---

## 🧱 Boas Práticas

- Ter baseline de configuração segura para cada stack.
- Separar configurações de dev, staging e produção.
- Não expor documentação interna sem autenticação.
- Usar IaC com revisão e validação de segurança.
- Criar checklist de deploy seguro.
- Revisar imagens Docker, manifests Kubernetes e API Gateway.
- Monitorar alterações de configuração sensíveis.
- Aplicar detecção de drift entre ambiente esperado e ambiente real.

---

## ✅ Checklist de Segurança — Misconfiguration

- [ ] CORS está restrito a origens confiáveis.
- [ ] Erros não exibem stack traces em produção.
- [ ] Swagger/OpenAPI está protegido ou desabilitado em produção.
- [ ] Actuator/debug/metrics possuem autenticação e menor exposição.
- [ ] Headers de segurança estão configurados.
- [ ] Métodos HTTP desnecessários estão bloqueados.
- [ ] Containers não rodam como root sem necessidade.
- [ ] Secrets não estão em código, imagem ou logs.
- [ ] Permissões cloud/IAM seguem menor privilégio.
- [ ] Configurações são revisadas em pipeline.

---

## 🔥 Resumo

> Configuração insegura também é vulnerabilidade.

Mesmo uma API com código correto pode ser comprometida se for publicada com CORS aberto, debug ativo, documentação exposta, permissões excessivas ou defaults inseguros.
