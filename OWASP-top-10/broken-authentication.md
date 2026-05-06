
---
tags: [security, owasp, api2, authentication, java, nodejs, pentest]
aliases:
  - API2 Broken Authentication
  - Autenticação Quebrada
  - Broken Authentication
---

# 🔐 API2:2023 — Autenticação Quebrada (Broken Authentication)

**Resumo:**  
A autenticação quebrada ocorre quando a API permite que invasores comprometam tokens, senhas, chaves de API ou fluxos de autenticação, permitindo o controle total de contas legítimas.  
Essas falhas geralmente decorrem de autenticação mal implementada, ausência de limitação de taxa, uso de tokens inseguros e verificação inadequada de identidade.

---

## 🔗 Associações no Obsidian

- Hub: [[API Security]]
- Tokens e credenciais: [[Vazamentos e Tokens de API]], [[JWT checklist]], [[Checklist ameaças OAuth 2.0]]
- Controles complementares: [[Unrestricted Resource Consumption|API4:2023 — Rate Limiting]], [[unrestricted-access-to-sensitive-business-flows|API6:2023 — Sensitive Business Flows]], [[Broken Object Level Authorization|BOLA]], [[broken-function-level-authorization|API5:2023 — BFLA]]
- Avaliação prática: [[Guia de consulta rápida — Avaliação REST]], [[Pentest]]

---

## 🧭 Vetores de Ataque

- Endpoints de autenticação acessíveis sem limitação de taxa.  
- Preenchimento de credenciais (credential stuffing) com listas de senhas conhecidas.  
- Tokens JWT com `"alg": "none"` ou chaves fracas.  
- Senhas armazenadas em texto puro ou com hash fraco.  
- Fluxos de redefinição de senha sem autenticação adequada.  
- Microsserviços acessíveis sem autenticação interna.  

---

## ⚙️ Fraquezas Comuns

- Falta de proteção contra força bruta (sem CAPTCHA ou bloqueio de conta).  
- Falta de expiração (`exp`) e verificação de validade (`iat`) em JWT.  
- Tokens não verificados ou aceitando assinaturas inválidas.  
- Permitir que usuários alterem e-mail/senha sem confirmar a senha atual.  
- Enviar credenciais pela URL (`GET /login?user=...&pass=...`).  
- Uso de chaves de API como autenticação de usuário.  

---

## 💥 Impactos

- Comprometimento de contas e dados sensíveis.  
- Ações sigilosas realizadas em nome de outros usuários.  
- Controle total da conta e possível movimentação lateral entre serviços.  

---

## Exemplo de Cenário #1 — Alteração de E-mail sem Confirmação

```node
PUT /account
Authorization: Bearer <token>
{
  "email": "attacker@evil.com"
}
```

🛡️ Mitigação correta (defesa em camadas)
🔐 1. Reautenticação obrigatória (step-up auth)

Antes de alterar o e-mail:

Solicitar senha atual OU MFA

```node
// Node.js (exemplo simplificado)
const bcrypt = require('bcrypt');

if (!bcrypt.compareSync(inputPassword, user.passwordHash)) {
  throw new Error('Reautenticação obrigatória');
}
```

👉 Isso bloqueia ataques com token roubado

---

📩 2. Confirmação via e-mail (double verification)

NÃO altere o e-mail diretamente

Fluxo correto:

Usuário solicita alteração
Sistema envia link para o novo e-mail
Opcional: notifica o e-mail antigo
Só após confirmação → efetiva mudança

```node
// pseudo fluxo
user.pendingEmail = "attacker@evil.com";
user.emailChangeToken = generateToken();

// envia link: /confirm-email-change?token=XYZ
```

⏳ 3. Delay + janela de reversão
Aguarde (ex: 5–30 minutos)
Permita cancelar via e-mail antigo

👉 reduz impacto mesmo se o atacante avançar

---

🚨 4. Notificação de segurança

Envie alerta para o e-mail atual:

“Seu e-mail está sendo alterado. Se não foi você, clique aqui para bloquear.”

---

**O invasor altera o e-mail da vítima e usa o fluxo “Esqueci minha senha” para sequestrar a conta.**

## 🧩 Exemplos de Código — Vulneráveis e Mitigados

### 🚨 Node.js (Express) — Exemplo Vulnerável

```node
// auth.js (vulnerável)
app.post('/login', async (req, res) => {
  const { username, password } = req.body;
  const user = await db.users.findOne({ username });
  if (!user || user.password !== password) { // comparação direta, sem hash
    return res.status(401).json({ error: 'Credenciais inválidas' });
  }
  const token = jwt.sign({ id: user.id }, 'weak-secret'); // chave fraca
  res.json({ token });
});
```

**Problemas:**

- Senha em texto puro.
- JWT com chave fraca e sem expiração.
- Sem limitação de tentativas de login.

### ✅ Node.js — Mitigação Recomendada

```node
// secure-auth.js
import rateLimit from 'express-rate-limit';
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';

const loginLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 3, // máximo de 3 tentativas por minuto
  message: { error: 'Muitas tentativas. Tente novamente mais tarde.' },
});

app.post('/login', loginLimiter, async (req, res) => {
  const { username, password } = req.body;
  const user = await db.users.findOne({ username });
  if (!user) return res.status(401).json({ error: 'Credenciais inválidas' });

  const valid = await bcrypt.compare(password, user.passwordHash);
  if (!valid) return res.status(401).json({ error: 'Credenciais inválidas' });

  const token = jwt.sign(
    { id: user.id, role: user.role },
    process.env.JWT_SECRET,
    { expiresIn: '15m', algorithm: 'HS256' }
  );

  res.json({ token });
});
```

**Melhorias:**

- Uso de `bcrypt` para hash de senha.
- `express-rate-limit` contra força bruta.
- JWT com chave segura e expiração curta.
- Segredo em variável de ambiente.

### 🚨 Java (Spring Boot) — Exemplo Vulnerável

```java
// AuthController.java (vulnerável)
@PostMapping("/login")
public ResponseEntity<?> login(@RequestBody UserLoginRequest request) {
    User user = userRepository.findByUsername(request.getUsername());
    if (user == null || !user.getPassword().equals(request.getPassword())) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body("Invalid credentials");
    }
    String token = Jwts.builder()
        .setSubject(user.getUsername())
        .signWith(SignatureAlgorithm.HS256, "123456") // chave fraca
        .compact();
    return ResponseEntity.ok(Map.of("token", token));
}
```

**Problemas:**

- Senhas armazenadas sem hash.
- Chave JWT previsível (“123456”).
- Sem expiração (`exp`).

### ✅ Java — Mitigação Recomendada

```java
// AuthController.java (seguro)
@PostMapping("/login")
public ResponseEntity<?> login(@RequestBody UserLoginRequest request) {
    User user = userRepository.findByUsername(request.getUsername());
    if (user == null || !passwordEncoder.matches(request.getPassword(), user.getPassword())) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body("Credenciais inválidas");
    }

    String token = Jwts.builder()
        .setSubject(user.getUsername())
        .claim("role", user.getRole())
        .setIssuedAt(new Date())
        .setExpiration(Date.from(Instant.now().plus(15, ChronoUnit.MINUTES)))
        .signWith(SignatureAlgorithm.HS256, jwtSecretKey)
        .compact();

    return ResponseEntity.ok(Map.of("token", token));
}
```

**Boas práticas aplicadas:**

- `BCryptPasswordEncoder` para hashing.
- Chave JWT armazenada de forma segura (`application.yml` ou Vault).
- Token com expiração e claims mínimos.
- Uso de roles e claims explícitos.

---

## 🧰 Recomendações Gerais (Checklist)

- [ ] Mapear todos os fluxos de autenticação (web, mobile, deep links).  
- [ ] Implementar **MFA** (autenticação multifator) sempre que possível.  
- [ ] Bloquear conta após número excessivo de tentativas falhas.  
- [ ] Validar expiração e integridade de tokens (`exp`, `iat`, `nbf`).  
- [ ] Não aceitar JWT com `alg: none`.  
- [ ] Aplicar hash de senhas usando `bcrypt`, `Argon2` ou `PBKDF2`.  
- [ ] Requerer senha atual para operações sensíveis (ex.: alteração de e-mail).  
- [ ] Não usar **chaves de API** para autenticação de usuários.  
- [ ] Evitar exposição de credenciais em **URLs, logs ou parâmetros de requisição**.  
- [ ] Usar bibliotecas padrão e frameworks testados (**Spring Security**, **Passport.js**, etc).  
- [ ] Adotar autenticação baseada em tokens robustos (**OAuth2/OpenID Connect**).  
- [ ] Tratar fluxos de **“Esqueci minha senha”** como endpoints de login (com rate-limit e bloqueio).  
