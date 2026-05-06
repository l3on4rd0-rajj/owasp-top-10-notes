
---
tags: [security, owasp, api3, authorization, java, nodejs, pentest]
aliases:
  - API3 BOPLA
  - Broken Object Property Level Authorization
  - Autorização Quebrada em Nível de Propriedade de Objeto
---

# 🔒 API3:2023 — Autorização Quebrada em Nível de Propriedade de Objeto (BOPLA)

**Resumo:**  
Esta vulnerabilidade ocorre quando uma API expõe ou permite a modificação de **propriedades internas/sensíveis** de um objeto que o usuário não deveria visualizar ou alterar.  
Trata-se de uma combinação das antigas falhas **Exposição Excessiva de Dados** e **Atribuição em Massa**.

---

## 🔗 Associações no Obsidian

- Hub: [[API Security]]
- Autorização: [[Broken Object Level Authorization|BOLA]], [[broken-function-level-authorization|API5:2023 — BFLA]], [[Guia de consulta rápida — Avaliação REST]]
- Dados sensíveis: [[Vazamentos e Tokens de API]], [[broken-authentication|API2:2023 — Autenticação Quebrada]]
- Testes ofensivos: [[Pentest]], [[Teste de Penetração vs Red Teaming]]

---

## 🧭 Vetores de Ataque

- Endpoints retornando objetos completos (ex.: `toJSON()`) sem filtro de propriedades.  
- Campos sensíveis expostos nas respostas (como `isAdmin`, `blocked`, `balance`, `price`, `token`, `passwordHash`).  
- Permitir modificações em campos que o cliente não deveria poder alterar.  
- Manipulação direta de propriedades via payload JSON.

---

## ⚙️ Fraquezas Comuns

- Falta de controle granular de propriedades acessíveis por usuário.  
- Uso de serialização automática sem filtragem (`to_json`, `toString`, `@Entity public`).  
- Falta de whitelists explícitas de campos que podem ser atualizados.  
- Endpoints reutilizados sem checagem de permissões por campo.  

---

## 💥 Impactos

- Exposição de informações privadas/sensíveis.  
- Modificação de atributos restritos, levando à fraude, manipulação de saldo, bypass de bloqueios ou escalonamento de privilégios.  
- Possível comprometimento parcial ou total de contas.  

---

## ⚠️ Exemplos de Cenário de Ataque

### 🧪 Cenário #1 — Exposição de Propriedades Sensíveis

Um endpoint retorna o objeto completo do usuário após uma denúncia:

```http
POST /api/reportUser
{
  "userId": 313,
  "reason": "offensive behavior"
}
```

Resposta:

```http
{
  "status": "ok",
  "reportedUser": {
    "id": 313,
    "fullName": "Alice Doe",
    "recentLocation": "São Paulo"
  }
}
```

➡️ **Problema:** Campos sensíveis (`fullName`, `recentLocation`) foram expostos indevidamente

### 🧪 Cenário #2 — Manipulação de Propriedade Restrita

O endpoint `/api/host/approve_booking` permite que o host aprove reservas:

```http

{
  "approved": true,
  "comment": "Check-in after 3pm"
}
```

O invasor modifica o payload:

```http

{
  "approved": true,
  "comment": "Check-in after 3pm",
  "total_stay_price": "1000000"
}
```

➡️ **Problema:** API não valida se `total_stay_price` pode ser alterado, permitindo superfaturamento.

```http

{
  "description": "funny cat video"
}
```

Invasor altera:

```http
{
  "description": "funny cat video",
  "blocked": false
}
```

➡️ **Problema:** Campo interno `blocked` pode ser alterado, desbloqueando conteúdo banido.


## 💻 Exemplos de Código — Vulnerável e Mitigado

### 🚨 Node.js (Express) — Exemplo Vulnerável

```node
// routes/video.js
router.put('/update', async (req, res) => {
  const { id, ...updateData } = req.body;
  const video = await Video.findByIdAndUpdate(id, updateData, { new: true }); // sem filtro
  res.json(video);
});
```

**Problemas:**

- Aceita qualquer propriedade enviada pelo cliente.
- Permite alteração de campos internos (ex.: `blocked`, `ownerId`).

### ✅ Node.js — Mitigação Recomendada

```node
// routes/video.js (seguro)
router.put('/update', async (req, res) => {
  const { id, description } = req.body; // whitelist explícita

  const video = await Video.findOne({ _id: id, owner: req.user.id });
  if (!video) return res.status(404).json({ error: 'Vídeo não encontrado' });

  video.description = description;
  await video.save();

  res.json({ id: video._id, description: video.description });
});
```

**Mitigação aplicada:**

- Whitelist de campos (`description`).
- Validação de propriedade (`owner`).
- Retorno controlado sem campos internos.

### 🚨 Java (Spring Boot) — Exemplo Vulnerável

```java
// VideoController.java
@PutMapping("/update")
public ResponseEntity<Video> updateVideo(@RequestBody Video video) {
    Video updated = videoRepository.save(video); // atribuição em massa
    return ResponseEntity.ok(updated);
}
```

**Problemas:**

- Permite alteração de todos os campos da entidade.
- Campos internos (ex.: `blocked`, `ownerId`, `price`) podem ser manipulados.

### ✅ Java — Mitigação Recomendada

```java
// VideoController.java (seguro)
@PutMapping("/update")
public ResponseEntity<?> updateVideo(@RequestBody VideoUpdateRequest req, Authentication auth) {
    User user = (User) auth.getPrincipal();
    Video video = videoRepository.findById(req.getId()).orElseThrow();

    if (!video.getOwner().getId().equals(user.getId())) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body("Acesso negado");
    }

    video.setDescription(req.getDescription()); // whitelist
    videoRepository.save(video);

    return ResponseEntity.ok(Map.of(
        "id", video.getId(),
        "description", video.getDescription()
    ));
}
```

Classe DTO segura:

```java
@Data
public class VideoUpdateRequest {
    private Long id;
    private String description; // somente campo permitido
}
```

**Mitigação aplicada:**

- Uso de DTOs para limitar campos modificáveis.
- Verificação de propriedade (`owner`).
- Exclusão de campos internos no retorno.

---

## 🧰 Como Prevenir

- [ ] Validar se o usuário tem permissão sobre **cada propriedade** do objeto acessado.  
- [ ] Evitar métodos genéricos como `to_json()` / `toString()` — retornar apenas os campos necessários.  
- [ ] Criar **DTOs (Data Transfer Objects)** com whitelists de propriedades expostas.  
- [ ] Evitar **atribuição em massa (mass assignment)** em frameworks ORM.  
- [ ] Permitir alterações apenas nos campos explicitamente autorizados.  
- [ ] Implementar validação de resposta baseada em **esquema** (JSON Schema ou Response Filter).  
- [ ] Manter os objetos retornados o mais **enxutos possível** — somente o essencial.  

---

## ✅ Checklist de Segurança — BOPLA

- [ ] Endpoints filtram as propriedades expostas conforme o papel do usuário.  
- [ ] Nenhum campo sensível é retornado (ex.: `password`, `token`, `blocked`, `price`).  
- [ ] Apenas propriedades permitidas podem ser alteradas via payload.  
- [ ] DTOs ou whitelists são usados em substituição a objetos diretos.  
- [ ] Nenhum endpoint utiliza `save(entity)` direto com objetos de requisição.  
- [ ] As respostas são validadas com base em um **esquema definido**.  
- [ ] Logs e auditorias registram tentativas de modificação indevida.  
- [ ] Testes automatizados validam que propriedades internas não são alteráveis.  
- [ ] Código revisado com foco em **mass assignment** e **exposição excessiva de dados**.  
