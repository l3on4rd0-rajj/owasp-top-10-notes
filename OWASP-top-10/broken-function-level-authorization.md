---
tags: [security, owasp, api5, authorization, access-control, java, nodejs, pentest]
aliases:
  - API5 Broken Function Level Authorization
  - Broken Function Level Authorization
  - BFLA
  - Autorização de Nível de Função Quebrada
---

# 🔐 API5:2023 — Autorização de Nível de Função Quebrada (BFLA)

**Resumo:**  
A **Autorização de Nível de Função Quebrada** ocorre quando a API não valida corretamente se um usuário tem permissão para executar uma ação ou acessar um endpoint específico.

Em outras palavras: o sistema autentica o usuário, mas não verifica se ele pode fazer aquilo que está tentando fazer.

---

## 🔗 Associações no Obsidian

- Hub: [[API Security]]
- Autorização: [[Broken Object Level Authorization|BOLA]], [[broken-object-property-level-authorization|BOPLA]]
- Abuso de fluxos: [[unrestricted-access-to-sensitive-business-flows|API6:2023 — Sensitive Business Flows]], [[Unrestricted Resource Consumption|API4:2023 — Rate Limiting]]
- Autenticação e identidade: [[broken-authentication|API2:2023 — Autenticação Quebrada]], [[JWT checklist]], [[Checklist ameaças OAuth 2.0]]
- Testes de API: [[Guia de consulta rápida — Avaliação REST]], [[Pentest]], [[Vulnerabilidades]]
- Resposta e evidência: [[Segurança de Aplicativos - Resposta a Incidentes]], [[GHAS]]

---

## 🧠 Descrição da Vulnerabilidade

Esse tipo de falha geralmente acontece quando:

- A aplicação confia apenas no fato do usuário estar autenticado.
- Não há verificação consistente de roles, grupos ou permissões.
- Endpoints administrativos não possuem proteção adequada.
- A segurança depende apenas do caminho da URL.
- O controle de autorização fica espalhado em cada controller, sem política central.

---

## ⚠️ A API é Vulnerável?

A melhor maneira de identificar esse problema é analisar o modelo de autorização:

- Um usuário comum pode acessar endpoints administrativos?
- Um usuário pode executar ações sensíveis alterando apenas o método HTTP, como `GET` para `DELETE`?
- Um usuário consegue acessar funções de outro grupo alterando apenas a URL?
- O sistema confia apenas na estrutura da URL para controle de acesso?
- Existe diferença real entre usuário autenticado, operador, gerente e administrador?

Nunca assuma que:

- `/api/admin` é seguro só porque parece administrativo.
- Endpoints administrativos estão sempre isolados.
- O front-end esconder um botão impede o acesso direto à função.

---

## 💣 Cenários de Ataque

### 🧨 Cenário #1 — Escalada para admin via endpoint oculto

Durante o cadastro, a aplicação usa:

```http
GET /api/invites/{invite_guid}
```

Um atacante altera a chamada para:

```http
POST /api/invites/new
```

➡️ **Problema:** o endpoint não valida autorização.

➡️ **Resultado:** criação indevida de convite ou usuário administrador.

### 🧨 Cenário #2 — Exposição de dados sensíveis

Endpoint administrativo exposto:

```http
GET /api/admin/v1/users/all
```

➡️ **Problema:** não existe validação de role.

➡️ **Resultado:** qualquer usuário autenticado consegue acessar dados de todos os usuários.

### 🧨 Cenário #3 — Alteração de método HTTP

Um usuário comum possui permissão apenas para consultar dados:

```http
GET /api/users/123
```

Mas tenta executar uma ação administrativa no mesmo recurso:

```http
DELETE /api/users/123
```

➡️ **Problema:** a API protege a rota de consulta, mas não valida autorização por função/ação.

➡️ **Resultado:** exclusão ou modificação indevida de recursos.

---

## 💻 Exemplos de Código — Vulnerável e Mitigado

### ❌ Node.js (Express) — Exemplo Vulnerável

```javascript
router.get('/admin/users', async (req, res) => {
  const users = await User.find();
  res.json(users);
});
```

**Problema:**

- Não verifica se o usuário é administrador.
- Confunde usuário autenticado com usuário autorizado.
- Expõe uma função administrativa diretamente.

### ✅ Node.js — Mitigação Recomendada

```javascript
function requireRole(role) {
  return (req, res, next) => {
    if (!req.user || req.user.role !== role) {
      return res.status(403).json({ error: 'Acesso negado' });
    }

    next();
  };
}

router.get('/admin/users', requireRole('admin'), async (req, res) => {
  const users = await User.find();
  res.json(users);
});
```

**Mitigação aplicada:**

- Middleware centralizado de autorização.
- Validação explícita de role.
- Bloqueio de acesso indevido com `403 Forbidden`.

### ❌ Java (Spring Boot) — Exemplo Vulnerável

```java
@GetMapping("/admin/users")
public List<User> getAllUsers() {
    return userRepository.findAll();
}
```

**Problema:**

- Nenhuma verificação de autorização.
- Qualquer usuário capaz de chamar o endpoint pode acessar a função.

### ✅ Java (Spring Security) — Mitigação Recomendada

```java
@PreAuthorize("hasRole('ADMIN')")
@GetMapping("/admin/users")
public List<User> getAllUsers() {
    return userRepository.findAll();
}
```

**Mitigação aplicada:**

- Controle baseado em role.
- Segurança declarativa.
- Autorização centralizada via Spring Security.

---

## 🛡️ Como Prevenir

- [ ] Implementar um módulo centralizado de autorização.
- [ ] Aplicar a política **negar tudo por padrão** e permitir explicitamente.
- [ ] Validar acesso com base em role, grupo, tenant, escopo e permissões específicas.
- [ ] Garantir que endpoints administrativos validem autorização no servidor.
- [ ] Proteger funções sensíveis mesmo que elas não apareçam no front-end.
- [ ] Testar todos os métodos HTTP permitidos para cada rota.
- [ ] Escrever testes negativos com usuários de baixo privilégio.
- [ ] Registrar tentativas de acesso negado para auditoria.

---

## 🧱 Boas Práticas

- Não confiar apenas em URL, método HTTP ou visibilidade no front-end.
- Separar autenticação de autorização.
- Centralizar políticas de acesso sempre que possível.
- Usar decorators, annotations, middlewares ou interceptors para padronizar controle.
- Revisar endpoints administrativos, rotas internas e funções de suporte.
- Validar autorização em cada ação sensível, não apenas no login.

---

## ✅ Checklist de Segurança — BFLA

- [ ] Cada endpoint possui regra clara de autorização.
- [ ] Usuários comuns não acessam funções administrativas.
- [ ] Métodos `POST`, `PUT`, `PATCH` e `DELETE` possuem proteção por permissão.
- [ ] Roles e permissões são validadas no back-end.
- [ ] Front-end não é usado como mecanismo de segurança.
- [ ] Testes cobrem acesso permitido e acesso negado.
- [ ] Logs registram tentativas de acesso indevido.
- [ ] A API retorna `403 Forbidden` para usuários autenticados sem permissão.

---

## 🔥 Resumo

> Autenticação não é autorização.

Se a API não valida **quem pode acessar cada função**, qualquer usuário autenticado pode executar ações que deveriam ser restritas.
