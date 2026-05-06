---
tags: [security, owasp, api6, business-flows, abuse, automation, bot, rate-limiting, pentest]
aliases:
  - API6 Unrestricted Access to Sensitive Business Flows
  - Unrestricted Access to Sensitive Business Flows
  - Acesso Irrestrito a Fluxos Sensíveis de Negócio
  - Abuso de Fluxos de Negócio
---

# 🚦 API6:2023 — Acesso Irrestrito a Fluxos Sensíveis de Negócio

**Resumo:**  
O **Acesso Irrestrito a Fluxos Sensíveis de Negócio** ocorre quando uma API permite o uso excessivo, automatizado ou abusivo de funções importantes para o negócio, mesmo que a função esteja tecnicamente funcionando como esperado.

Essa falha não depende necessariamente de autenticação quebrada ou autorização ausente. Muitas vezes, o usuário tem permissão para executar a ação, mas a API não controla **como**, **quantas vezes**, **em qual ritmo** ou **com qual intenção** essa ação pode ser executada.

---

## 🔗 Associações no Obsidian

- Hub: [[API Security]]
- Abuso e limites: [[Unrestricted Resource Consumption|API4:2023 — Rate Limiting]], [[broken-authentication|API2:2023 — Autenticação Quebrada]]
- Autorização: [[broken-function-level-authorization|API5:2023 — BFLA]], [[Broken Object Level Authorization|BOLA]]
- Testes de API: [[Guia de consulta rápida — Avaliação REST]], [[Pentest]], [[Vulnerabilidades]]
- Resposta e operação: [[Segurança de Aplicativos - Resposta a Incidentes]], [[GHAS]]

---

## 🧠 Descrição da Vulnerabilidade

Essa vulnerabilidade aparece quando fluxos críticos do negócio podem ser automatizados ou abusados sem controles adequados.

Exemplos comuns:

- Comprar todos os ingressos disponíveis antes de usuários legítimos.
- Reservar produtos, assentos, horários ou recursos em massa.
- Criar várias contas para fraude, spam ou abuso de promoções.
- Usar cupons, bônus ou créditos repetidamente.
- Automatizar tentativas de recuperação de senha, OTP ou convites.
- Fazer scraping massivo de dados expostos por endpoints legítimos.
- Enviar avaliações, votos, mensagens ou solicitações em escala.

---

## ⚠️ A API é Vulnerável?

A API pode estar vulnerável se:

- Fluxos sensíveis não possuem proteção contra automação.
- Não há limite por usuário, IP, conta, dispositivo, sessão ou método de pagamento.
- A API aceita chamadas diretas sem validar o comportamento esperado do fluxo.
- A aplicação confia apenas no front-end para impor etapas do processo.
- Não há detecção de padrões anormais de uso.
- Ações de alto impacto não exigem confirmação, fricção ou validação adicional.
- O mesmo usuário consegue repetir uma operação sensível em escala.

Perguntas úteis:

- Este endpoint pode gerar lucro, prejuízo ou vantagem competitiva se automatizado?
- Uma pessoa real executaria essa ação nesse volume e velocidade?
- O fluxo tem controles por etapa ou apenas valida o resultado final?
- Existe monitoramento de abuso por comportamento, não apenas por erro?

---

## 💣 Cenários de Ataque

### 🧨 Cenário #1 — Compra automatizada de ingressos

Uma API permite reservar ingressos:

```http
POST /api/tickets/reserve
{
  "eventId": "show-123",
  "quantity": 4
}
```

Um atacante automatiza milhares de requisições com várias contas.

➡️ **Problema:** a API não limita reservas por conta, IP, dispositivo ou método de pagamento.

➡️ **Resultado:** usuários legítimos ficam sem ingressos, e o atacante revende os bilhetes.

### 🧨 Cenário #2 — Abuso de cupom promocional

Endpoint de aplicação de cupom:

```http
POST /api/checkout/apply-coupon
{
  "coupon": "WELCOME10"
}
```

O atacante cria muitas contas e reutiliza a promoção repetidamente.

➡️ **Problema:** a regra de negócio é validada por conta, mas não por dispositivo, cartão, endereço, telefone ou padrão de criação.

➡️ **Resultado:** prejuízo financeiro e distorção da campanha promocional.

### 🧨 Cenário #3 — Enumeração e scraping de dados

Endpoint legítimo de consulta:

```http
GET /api/products?page=1&limit=100
```

O atacante percorre milhares de páginas em alta velocidade para extrair catálogo, preços ou dados sensíveis.

➡️ **Problema:** não há limitação comportamental nem detecção de scraping.

➡️ **Resultado:** extração massiva de dados de negócio.

### 🧨 Cenário #4 — Fluxo de OTP sem controle de abuso

Endpoint de envio de OTP:

```http
POST /api/auth/send-otp
{
  "phone": "+5511999999999"
}
```

O atacante dispara muitos códigos para o mesmo número ou para uma lista de números.

➡️ **Problema:** não há cooldown, limite por destino ou controle de custo.

➡️ **Resultado:** spam, custo financeiro e degradação da experiência do usuário.

---

## 💻 Exemplos de Código — Vulnerável e Mitigado

### ❌ Node.js (Express) — Exemplo Vulnerável

```javascript
router.post('/checkout/apply-coupon', async (req, res) => {
  const { coupon } = req.body;
  const discount = await couponService.apply(req.user.id, coupon);

  res.json({ discount });
});
```

**Problemas:**

- Não valida limite de uso por campanha.
- Não avalia repetição por dispositivo, cartão, telefone ou endereço.
- Não detecta criação massiva de contas para abuso da promoção.

### ✅ Node.js — Mitigação Recomendada

```javascript
router.post('/checkout/apply-coupon', async (req, res) => {
  const { coupon } = req.body;

  const allowed = await abuseControl.canUseCoupon({
    userId: req.user.id,
    coupon,
    ip: req.ip,
    deviceId: req.headers['x-device-id'],
    paymentFingerprint: req.body.paymentFingerprint,
  });

  if (!allowed) {
    return res.status(429).json({ error: 'Limite de uso atingido' });
  }

  const discount = await couponService.apply(req.user.id, coupon);
  res.json({ discount });
});
```

**Mitigação aplicada:**

- Controle por usuário, IP, dispositivo e pagamento.
- Regra de negócio aplicada no back-end.
- Resposta com `429 Too Many Requests` quando houver abuso.

### ❌ Java (Spring Boot) — Exemplo Vulnerável

```java
@PostMapping("/tickets/reserve")
public Reservation reserve(@RequestBody ReservationRequest req, Authentication auth) {
    return reservationService.reserve(auth.getName(), req.getEventId(), req.getQuantity());
}
```

**Problemas:**

- Não controla volume por usuário.
- Não impede múltiplas reservas simultâneas.
- Não aplica regra antifraude antes da reserva.

### ✅ Java (Spring Boot) — Mitigação Recomendada

```java
@PostMapping("/tickets/reserve")
public ResponseEntity<?> reserve(@RequestBody ReservationRequest req, Authentication auth) {
    String userId = auth.getName();

    if (!abuseControl.canReserve(userId, req.getEventId(), req.getQuantity())) {
        return ResponseEntity.status(429).body("Limite de reserva atingido");
    }

    Reservation reservation = reservationService.reserve(
        userId,
        req.getEventId(),
        req.getQuantity()
    );

    return ResponseEntity.ok(reservation);
}
```

**Mitigação aplicada:**

- Validação de limite antes da ação sensível.
- Controle específico para o fluxo de reserva.
- Redução de abuso automatizado.

---

## 🛡️ Como Prevenir

- [ ] Identificar fluxos sensíveis para o negócio.
- [ ] Definir limites por usuário, IP, sessão, dispositivo, conta, telefone, e-mail e método de pagamento.
- [ ] Implementar rate limiting contextual, não apenas global.
- [ ] Validar etapas do fluxo no servidor.
- [ ] Usar detecção de comportamento anormal e automação.
- [ ] Adicionar fricção progressiva, como CAPTCHA, MFA ou confirmação adicional, quando houver risco.
- [ ] Registrar eventos de abuso com contexto suficiente para investigação.
- [ ] Aplicar alertas para picos incomuns de uso.
- [ ] Proteger endpoints de alto custo financeiro ou operacional.
- [ ] Revisar campanhas, promoções, reservas e recursos limitados com visão antifraude.

---

## 🧱 Boas Práticas

- Não confiar no front-end para limitar ações sensíveis.
- Pensar em abuso de negócio, não apenas em bugs técnicos.
- Tratar automação excessiva como risco de segurança.
- Aplicar limites específicos por fluxo.
- Monitorar padrões de uso legítimo versus uso automatizado.
- Implementar controles progressivos em vez de bloquear tudo de forma fixa.
- Usar logs, métricas e alertas para detectar abuso em tempo real.

---

## ✅ Checklist de Segurança — Business Flows

- [ ] Fluxos críticos foram mapeados.
- [ ] Endpoints sensíveis possuem limites por contexto.
- [ ] A API bloqueia repetição abusiva de ações.
- [ ] Promoções e cupons possuem controle antifraude.
- [ ] Reservas, compras e recursos escassos possuem limites claros.
- [ ] OTP, reset de senha e convites possuem cooldown e limite por destino.
- [ ] Scraping e enumeração são monitorados.
- [ ] Eventos suspeitos geram logs e alertas.
- [ ] Existe resposta operacional para abuso em andamento.

---

## 🔥 Resumo

> Nem todo abuso de API parece uma falha técnica.

Se um fluxo sensível pode ser automatizado em escala sem controle, a API pode permitir fraude, exaustão de recursos, perda financeira ou vantagem indevida.
