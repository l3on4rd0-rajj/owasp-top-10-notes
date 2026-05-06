---
tags: [security, owasp, api9, inventory, asset-management, api-catalog, shadow-api, deprecated-api, devsecops]
aliases:
  - API9 Improper Inventory Management
  - Improper Inventory Management
  - Gestão Inadequada de Inventário
  - Shadow API
  - API Inventory
---

# 🗂️ API9:2023 — Gestão Inadequada de Inventário

**Resumo:**  
**Gestão Inadequada de Inventário** ocorre quando a organização não possui visibilidade clara sobre suas APIs, versões, endpoints, ambientes, donos, dados expostos e dependências.

Sem inventário atualizado, APIs antigas, esquecidas ou não documentadas continuam expostas, muitas vezes sem os mesmos controles de autenticação, autorização, logging, rate limiting e hardening das APIs principais.

---

## 🔗 Associações no Obsidian

- Hub: [[API Security]]
- Gestão e hardening: [[security-misconfiguration|API8:2023 — Security Misconfiguration]], [[GHAS]], [[github]]
- Testes e descoberta: [[Guia de consulta rápida — Avaliação REST]], [[Pentest]], [[Vulnerabilidades]]
- Infra e ambientes: [[Containers e Orquestração]], [[docker compose]], [[minikube]]
- Resposta e operação: [[Segurança de Aplicativos - Resposta a Incidentes]]

---

## 🧠 Descrição da Vulnerabilidade

Esse risco surge quando a organização não sabe exatamente:

- Quais APIs existem.
- Quais endpoints estão expostos.
- Quais versões ainda estão em produção.
- Quem é o dono técnico e de negócio de cada API.
- Quais dados cada API processa.
- Quais ambientes estão acessíveis publicamente.
- Quais dependências, integrações e consumidores usam cada endpoint.
- Quais APIs estão obsoletas, duplicadas ou sem manutenção.

O resultado é uma superfície de ataque maior e difícil de proteger.

---

## ⚠️ A API é Vulnerável?

A API pode estar vulnerável se:

- Existem versões antigas como `/v1`, `/v2`, `/beta`, `/legacy` ainda acessíveis.
- A documentação não reflete o que está em produção.
- Não existe catálogo central de APIs.
- Ambientes de teste, staging ou homologação estão públicos.
- APIs internas estão expostas sem necessidade.
- Endpoints não possuem dono claro.
- APIs depreciadas continuam recebendo tráfego.
- Equipes não sabem quais dados sensíveis passam por cada endpoint.
- Não há processo de aposentadoria de APIs antigas.

Sinais comuns:

- Rotas esquecidas em gateways.
- Subdomínios antigos ainda resolvendo.
- Swagger antigo exposto.
- Serviços abandonados com autenticação fraca.
- Diferentes versões com regras de segurança inconsistentes.

Exemplos de padrões que devem aparecer no inventário:

```text
/api/v1/*
/api/v2/*
/api/beta/*
/api/legacy/*
/internal/*
/admin/*
/swagger.json
/v3/api-docs
```

---

## 💣 Cenários de Ataque

### 🧨 Cenário #1 — API antiga sem controle moderno

API atual:

```http
GET /api/v3/users/me
Authorization: Bearer <token>
Host: api.example.com
```

Versão antiga ainda exposta:

```http
GET /api/v1/users/123
Host: api.example.com
```

➡️ **Problema:** a versão `v1` não aplica as mesmas regras de autorização da versão atual.

➡️ **Resultado:** acesso indevido a dados de outros usuários.

### 🧨 Cenário #2 — Swagger antigo exposto

Endpoint esquecido:

```http
GET /api-old/swagger.json
Host: api.example.com
```

➡️ **Problema:** a documentação antiga revela endpoints, parâmetros e modelos internos.

➡️ **Resultado:** o atacante usa a especificação para mapear rotas que ninguém monitora.

### 🧨 Cenário #3 — Ambiente de homologação público

Subdomínio antigo:

```text
https://staging-api.example.com
https://homolog-api.example.com
https://dev-api.example.com
```

➡️ **Problema:** o ambiente possui dados reais, autenticação fraca e configurações de debug.

➡️ **Resultado:** vazamento de dados e exploração de endpoints menos protegidos.

### 🧨 Cenário #4 — Shadow API fora do gateway

API principal passa pelo gateway:

```text
https://api.example.com/orders
```

Serviço direto exposto:

```text
https://orders-service.example.com/internal/orders
```

➡️ **Problema:** o serviço direto não passa por autenticação central, WAF, rate limiting ou logging.

➡️ **Resultado:** bypass dos controles aplicados no gateway.

---

## 💻 Exemplos de Código — Vulnerável e Mitigado

### ❌ Node.js (Express) — Exemplo Vulnerável

```javascript
app.use('/api/v1/users', legacyUsersRouter);
app.use('/api/v2/users', usersRouter);
app.use('/api/admin', adminRouter);
```

**Problemas:**

- Versão legada continua ativa sem plano de desativação.
- Não há metadados de dono, criticidade ou status.
- Rotas antigas podem ter políticas de segurança diferentes.

### ✅ Node.js — Mitigação Recomendada

Exemplo de inventário simples mantido junto à aplicação:

```javascript
const apiInventory = [
  {
    path: '/api/v2/users',
    owner: 'identity-team',
    status: 'active',
    dataClass: 'personal-data',
    authRequired: true,
  },
  {
    path: '/api/v1/users',
    owner: 'identity-team',
    status: 'deprecated',
    sunsetAt: '2026-06-30',
    authRequired: true,
  },
];

function blockRetiredApis(req, res, next) {
  const route = apiInventory.find(item => req.path.startsWith(item.path));

  if (route?.status === 'retired') {
    return res.status(410).json({ error: 'API descontinuada' });
  }

  next();
}

app.use(blockRetiredApis);
app.use('/api/v2/users', usersRouter);
```

O mesmo conceito pode ser representado em YAML para revisão em pipeline:

```yaml
apis:
  - path: /api/v2/users
    owner: identity-team
    status: active
    dataClass: personal-data
    authRequired: true

  - path: /api/v1/users
    owner: identity-team
    status: deprecated
    sunsetAt: 2026-06-30
    authRequired: true
```

**Mitigação aplicada:**

- Inventário explícito de rotas.
- Status de ciclo de vida.
- Bloqueio para APIs aposentadas.
- Base para governança e revisão contínua.

### ❌ Java (Spring Boot) — Exemplo Vulnerável

```java
@RestController
@RequestMapping("/api/v1/users")
public class LegacyUserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userRepository.findById(id).orElseThrow();
    }
}
```

**Problemas:**

- Endpoint legado sem dono visível.
- Não há indicação de depreciação.
- Pode retornar entidade completa sem filtro.

### ✅ Java (Spring Boot) — Mitigação Recomendada

Endpoint legado respondendo explicitamente como descontinuado:

```java
@Deprecated
@RestController
@RequestMapping("/api/v1/users")
public class LegacyUserController {

    @GetMapping("/{id}")
    public ResponseEntity<?> getUser(@PathVariable Long id) {
        return ResponseEntity
            .status(HttpStatus.GONE)
            .body("API descontinuada. Use /api/v2/users.");
    }
}
```

Nova versão orientada ao usuário autenticado:

```java
@RestController
@RequestMapping("/api/v2/users")
public class UserController {

    @GetMapping("/me")
    public UserDto getCurrentUser(Authentication auth) {
        return userService.findSafeProfile(auth.getName());
    }
}
```

**Mitigação aplicada:**

- API antiga explicitamente descontinuada.
- Retorno `410 Gone` para rota aposentada.
- Nova versão usa endpoint mais seguro e orientado ao usuário autenticado.

---

## 🛡️ Como Prevenir

- [ ] Manter catálogo central de APIs, endpoints, versões e ambientes.
- [ ] Registrar dono técnico e dono de negócio de cada API.
- [ ] Classificar dados processados por cada endpoint.
- [ ] Documentar consumidores, integrações e dependências.
- [ ] Ter processo formal de versionamento, depreciação e aposentadoria.
- [ ] Remover versões antigas que não são mais necessárias.
- [ ] Garantir que APIs internas não fiquem expostas publicamente.
- [ ] Revisar subdomínios, gateways, ingress, load balancers e DNS.
- [ ] Monitorar tráfego para endpoints obsoletos.
- [ ] Validar inventário com descoberta automatizada e testes recorrentes.

---

## 🧱 Boas Práticas

- Usar API Gateway como ponto central de controle e visibilidade.
- Padronizar OpenAPI/Swagger e manter especificações versionadas.
- Vincular APIs a owners claros.
- Aplicar tags de criticidade, ambiente e classificação de dados.
- Automatizar descoberta de subdomínios e endpoints.
- Comparar documentação com tráfego real.
- Criar política de sunset para versões antigas.
- Garantir que staging/homologação não use dados reais sem proteção.
- Integrar inventário com CI/CD e ferramentas de segurança.

---

## ✅ Checklist de Segurança — Inventory Management

- [ ] Todas as APIs conhecidas estão catalogadas.
- [ ] Cada API tem owner técnico e de negócio.
- [ ] Versões antigas possuem data de sunset.
- [ ] APIs aposentadas retornam `410 Gone` ou foram removidas.
- [ ] Documentação OpenAPI está atualizada.
- [ ] Ambientes de dev/staging não estão públicos sem proteção.
- [ ] Endpoints internos não contornam gateway, WAF ou autenticação central.
- [ ] Dados sensíveis por endpoint estão classificados.
- [ ] Tráfego para rotas legadas é monitorado.
- [ ] Descoberta automatizada identifica shadow APIs.

---

## 🔥 Resumo

> Não dá para proteger o que não se conhece.

Sem inventário atualizado, APIs antigas, esquecidas ou paralelas podem permanecer expostas com controles fracos, tornando-se caminhos fáceis para vazamento, abuso e bypass de segurança.
