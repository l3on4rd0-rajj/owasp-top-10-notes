---
title: "Checklist de Segurança — JSON Web Token (JWT)"
tags: [security, jwt, checklist, pentest]
---

## 🔗 Associações no Obsidian

- Hub: [[API Security]]
- Autenticação: [[broken-authentication|Broken Authentication]], [[Checklist ameaças OAuth 2.0]]
- Segredos: [[Vazamentos e Tokens de API]], [[GHAS]]
- Testes: [[Guia de consulta rápida — Avaliação REST]], [[Pentest]]

---
---
title: "Checklist de Segurança — JSON Web Token (JWT)"
tags: [security, jwt, checklist, pentest]
---

# ✅ Checklist de Segurança — JSON Web Token (JWT)

---

## 🔎 Revisão do Header
- [ ] Verificar a estrutura do JWT: **header.payload.signature** (cada parte em _URL-safe Base64_ sem padding).
- [ ] Confirmar que o campo `typ` (tipo) e `alg` (algoritmo) estão corretos e previstos.
- [ ] Aplicar **whitelist** de algoritmos (aceitar apenas algoritmos permitidos).
- [ ] Desabilitar suporte ao algoritmo `"none"` (não aceitar tokens sem assinatura).
- [ ] Verificar se não há **injeção** no elemento `kid` (key id).
- [ ] Tratar como **não confiável** qualquer `jwk` embutido (não confiar cegamente em chaves JWK fornecidas pelo cliente).
- [ ] Garantir que chaves/segredos sejam **diferentes entre ambientes** (dev / staging / prod).

---

## 🧾 Revisão do Payload
- [ ] Revisar o payload em busca de **informação sensível** armazenada (evitar PII, segredos, tokens, senhas).
- [ ] Verificar presença/uso correto de claims essenciais (ex.: `sub`, `aud`, `iss`).
- [ ] Não confiar em dados do payload sem validação no servidor (payload é facilmente decodificável).

---

## 🔐 Revisão da Assinatura
- [ ] Confirmar que a **assinatura é verificada** pelo servidor (não apenas decodificada).
- [ ] Verificar que **chaves e segredos** estão armazenados **fora do código-fonte** (variáveis de ambiente seguras, cofres de segredos).
- [ ] Testar (em ambiente autorizado) força/brute force contra chaves secretas fracas para avaliar força/complexidade.
- [ ] Para algoritmos HMAC, usar verificação com **comparação em tempo constante** (time-constant verification).
- [ ] Garantir que as chaves/segredos usados em cada ambiente **não sejam os mesmos**.

---

## 🔁 Proteções e Validações Adicionais
- [ ] Implementar proteção contra **replay** (usar `jti` e manter lista/blacklist de tokens revogados quando aplicável).
- [ ] Enforce de expiração do token — validar `exp` (expiration).
- [ ] Validar também `iat` (issued at) e, se aplicável, `nbf` (not before).
- [ ] Garantir que todas as rotas que exigem autenticação **exijam checagem de assinatura**.
- [ ] Validar `aud` (audience) e `iss` (issuer) conforme esperado pela aplicação.

---

## 🛡️ Boas práticas de armazenamento e operação
- [ ] **Armazenar segredos/chaves fora do repositório** (ex.: HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, variáveis de ambiente protegidas).
- [ ] Rotacionar chaves/segredos periodicamente e ter plano de revogação.
- [ ] Usar algoritmos e tamanhos de chave recomendados e atualizados (ex.: RS256/ES256 para assinaturas públicas; HMAC com chave forte se necessário).
- [ ] Desabilitar suportes/algoritmos obsoletos ou inseguros.

---

## 📝 Observações técnicas rápidas
- [ ] Lembrar: JWT = `header.payload.signature` (cada parte em Base64 URL-safe sem padding).
- [ ] Não confiar apenas em decodificação para autenticação — sempre validar assinatura e claims.
- [ ] Exemplos (apenas para inspeção):
  - `eyJ0eXAiOiJKV1QiLCJh...` → header (Base64 URL-safe)
  - `eyJsb2dpbiI6ImFkbWluIn0` → payload (Base64 URL-safe)
  - `FSfvCBAwypJ4abF6jFLmR7...` → signature

---

## 🔗 Referência / Fonte
- [ ] Checklist adaptado do material técnico (use apenas em **testes autorizados** — pentests e auditorias com permissão).
- [ ] Fonte original de estudo: Pentester Lab / @PentesterLab.

---

### ✨ Dica final
- [ ] Sempre aplique essas verificações em um **ambiente de testes autorizado**. Não realize ataques (brute force, injeções, etc.) em sistemas sem permissão explícita.
