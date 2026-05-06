---
tags: [security, bola, authorization, api, jwt, spring, nodejs]
aliases:
  - BOLA
  - API1 Broken Object Level Authorization
  - Autorização em nível de objeto
---

# 🔐 BOLA — Broken Object Level Authorization (Autorização em nível de objeto)

**Resumo:**
Invasores exploram endpoints que recebem um *object id* (ID do recurso) e executam ações sem validar se o usuário autenticado tem permissão sobre aquele objeto. IDs previsíveis (sequenciais) facilitam a enumeração; ausência de checagem de propriedade permite leitura/alteração/exclusão de recursos de outros usuários.

---

## 🔗 Associações no Obsidian

- Hub: [[API Security]]
- Autorização e exposição: [[broken-object-property-level-authorization|API3:2023 — BOPLA]], [[broken-function-level-authorization|API5:2023 — BFLA]], [[Guia de consulta rápida — Avaliação REST]]
- Identidade e sessão: [[broken-authentication|API2:2023 — Autenticação Quebrada]], [[JWT checklist]]
- Prática ofensiva: [[Pentest]], [[1.1 O que é Red Teaming]]

---

## 🧭 Vetores de ataque (resumido)

- IDs de objetos expostos em URLs, querystrings, cabeçalhos ou payloads.
- IDs previsíveis (1,2,3...).
- Endpoints que apenas confiam no ID enviado pelo cliente para acessar/alterar recursos.
- Falta de verificação "o usuário X é proprietário / tem permissão sobre o objeto Y?"

---

## ⚠️ Impactos

- Divulgação de dados confidenciais de terceiros.
- Modificação/exclusão de dados de outros usuários.
- Escalada até takeover de contas (em certos fluxos).

---

## ✅ Regras gerais de prevenção

- **Sempre** valide autorização em nível de objeto: após obter o recurso, verifique que o usuário autenticado tem permissão para agir sobre ele.
- Use IDs **não previsíveis** (UUIDs/GUIDs ou IDs randomizados) para dificultar enumeração.
- Aplique autorização centralizada (middleware / interceptors / AOP / method security).
- Não confie apenas em informações enviadas pelo cliente (ex.: `ownerId` no payload).
- Escreva testes automatizados que tentem acessar objetos de outros usuários (positive & negative tests).

---

## 📌 Exemplos de cenários (resumidos)

- Endpoint REST: `GET /shops/{shopName}/revenue_data.json` — sem verificação, permite acesso a dados de qualquer loja.
- API de veículo usando `VIN` sem verificar vínculo com usuário — permite controlar veículos de terceiros.
- GraphQL mutation `deleteReports(reportKeys: [...])` — exclui documentos sem validar permissões.

---

## 🧩 Exemplos de código — *Vulnerável* e *Mitigado*

> **OBS:** exemplos didáticos. Em produção, adapte a integração com seu sistema de autenticação/identidade, logging e tratamento de erros.

---

### Java — Spring Boot

#### Vulnerável

```java
// ShopController.java (Vulnerável)
@RestController
@RequestMapping("/shops")
public class ShopController {

    @Autowired
    private ShopService shopService;

    @GetMapping("/{shopName}/revenue_data.json")
    public ResponseEntity<RevenueDto> getRevenue(@PathVariable String shopName) {
        // NÃO há verificação de autorização por proprietário/tenant
        RevenueDto revenue = shopService.getRevenueByShopName(shopName);
        if (revenue == null) return ResponseEntity.notFound().build();
        return ResponseEntity.ok(revenue);
    }
}
```

**Problema:** qualquer usuário pode solicitar `/shops/anyShop/revenue_data.json` e obter dados.

#### Mitigação 1 — Verificar propriedade no controller

```java
// ShopController.java (Mitigado - verificação manual)
@RestController
@RequestMapping("/shops")
public class ShopController {

    @Autowired
    private ShopService shopService;

    @GetMapping("/{shopName}/revenue_data.json")
    public ResponseEntity<RevenueDto> getRevenue(
            @PathVariable String shopName,
            @AuthenticationPrincipal UserPrincipal user // extraído do token/session
    ) {
        Shop shop = shopService.findByName(shopName);
        if (shop == null) return ResponseEntity.notFound().build();

        // Verifica se o usuário autenticado tem acesso (ex.: é dono ou admin)
        if (!shopService.userHasAccess(user.getId(), shop)) {
            return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
        }

        RevenueDto revenue = shopService.getRevenueByShop(shop);
        return ResponseEntity.ok(revenue);
    }
}
```

#### Mitigação 2 — Usar autorização declarativa (Spring Security / Method Security)

```java

// ShopService.java
@Service
public class ShopService {
    // método protegido por @PreAuthorize
    @PreAuthorize("@shopSecurity.canAccessShop(#shopName, principal)")
    public RevenueDto getRevenueByShopName(String shopName) {
        // ...
    }
}

// ShopSecurity.java (bean)
@Component
public class ShopSecurity {
    public boolean canAccessShop(String shopName, UserPrincipal principal) {
        // lógica: verificar se principal.id está entre os donos/tenants da loja
        // ou se principal.hasRole("ADMIN")
        return // true/false;
    }
}
```

**Vantagem: centraliza verificação, reduz chance de esquecer checagem.**

---

### Node.js — Express + Mongoose (MongoDB)

#### Vulnerável

```node
// routes/documents.js (Vulnerável)
const express = require('express');
const router = express.Router();
const Document = require('../models/Document');

router.post('/delete', async (req, res) => {
  const docId = req.body.documentId;
  // Exclusão direta sem verificação de propriedade
  await Document.findByIdAndDelete(docId);
  return res.json({ ok: true });
});

module.exports = router;
```

**Problema:** um usuário pode enviar `documentId` de outro usuário e excluí-lo.

#### Mitigação 1 — Verificar propriedade/autor no handler

```node
// middleware/auth.js (exemplo de middleware que popula req.user)
const jwt = require('jsonwebtoken');
module.exports = function (req, res, next) {
  // extrai e valida token, popula req.user (id, roles, etc.)
  // ...
  next();
};

// routes/documents.js (Mitigado)
const express = require('express');
const router = express.Router();
const Document = require('../models/Document');
const ensureAuth = require('../middleware/auth');

router.post('/delete', ensureAuth, async (req, res) => {
  const docId = req.body.documentId;
  const doc = await Document.findById(docId);
  if (!doc) return res.status(404).send({ error: 'Document not found' });

  // Checa se o usuário é proprietário ou tem permissão
  if (doc.owner.toString() !== req.user.id && !req.user.roles.includes('admin')) {
    return res.status(403).send({ error: 'Forbidden' });
  }

  await doc.remove();
  return res.json({ ok: true });
});

module.exports = router;
```

#### Mitigação 2 — Query segura que combina ID do recurso e do dono

```node
router.post('/delete', ensureAuth, async (req, res) => {
  const docId = req.body.documentId;
  // Remove apenas se encontrar o documento com owner igual ao usuário
  const result = await Document.findOneAndDelete({ _id: docId, owner: req.user.id });
  if (!result) return res.status(404).send({ error: 'Not found or not allowed' });
  return res.json({ ok: true });
});
```

---

## 🔁 Boas práticas adicionais / dicas rápidas

- **Preferir queries que já filtram por owner** (ex.: `DELETE FROM resource WHERE id = ? AND owner_id = ?`).
- **Logs de auditoria**: registre tentativas de acesso negadas para detecção e investigação.
- **Rate-limit / monitoring**: detectar enumeração de IDs (vários 404/403 sequenciais).
- **IDs não sequenciais**: use UUIDv4 ou identificadores curtos randomizados (ex.: ULID) para reduzir sucesso de enumeração automática.
- **Least privilege**: usuários só devem ter permissões mínimas necessárias.

## ✅ Checklist rápido (copiar/colar)

- [ ] Cada endpoint que recebe ID valida autorização em nível de objeto.
- [ ] Não confiar em `ownerId` enviado pelo cliente.
- [ ] Preferir queries que combinam `id` + `owner_id` em uma única operação.
- [ ] Usar IDs não previsíveis quando possível.
- [ ] Adicionar testes automatizados cobrindo tentativas de acesso por usuários não proprietários.
- [ ] Monitorar padrões de enumeração (muitos 404/403 sequenciais).
- [ ] Registrar (audit) tentativas de acesso negadas.
