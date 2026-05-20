
---
tags: [api, rest, pentest, appsec, fuzzing, checklist]
aliases:
  - Avaliação REST
  - REST Security Testing
  - Guia REST
---

## 🔗 Associações no Obsidian

- Hub: [[API Security]]
- Testes ofensivos: [[Operações Ofensivas]], [[Pentest]], [[Vulnerabilidades]]
- OWASP API: [[Broken Object Level Authorization|BOLA]], [[broken-object-property-level-authorization|BOPLA]], [[broken-authentication|Broken Authentication]], [[Unrestricted Resource Consumption|Rate Limiting]], [[broken-function-level-authorization|Broken Function Level Authorization]], [[unrestricted-access-to-sensitive-business-flows|Sensitive Business Flows]], [[server-side-request-forgery|SSRF]], [[security-misconfiguration|Security Misconfiguration]], [[improper-inventory-management|Improper Inventory Management]]
- Tokens e sessão: [[JWT checklist]], [[Checklist ameaças OAuth 2.0]], [[Vazamentos e Tokens de API]]

---

## 🔹 Sobre os serviços web RESTful
Os serviços web são usados para comunicação entre máquinas (apps web, mobile, desktop).  
**RESTful** é a variante leve que usa verbos HTTP (GET, POST, PUT, DELETE) em vez de protocolos complexos como SOAP.

---

## 🔑 Principais propriedades
- **Verbos HTTP**: GET, POST, PUT, DELETE (entre outros).  
- **Parâmetros não padronizados**:
  - URL path
  - Query string
  - Headers HTTP (ex.: `X-Custom-Id`)
  - Corpo (JSON / XML)
- **Formato**: JSON ou XML (valores, corpos, parâmetros estruturados).  
- **Autenticação**: tokens (JWT, API keys, HMAC, schemes customizados).  
- **Documentação**: rara; WADL existe, mas pouco adotado — procurar README / guia do dev.

---

## ⚠️ Desafios nos testes de segurança
- UI frequentemente **não revela** toda a superfície de ataque.
- Recursos ativados dinamicamente (cliente JS / apps móveis).
- Parâmetros espalhados entre path / headers / body.
- Estruturas JSON grandes aumentam o custo do fuzzing.
- Autenticação customizada exige engenharia reversa.

---

## 🛠️ Como testar — passos práticos

### 1) Determinar a superfície de ataque
- Buscar documentação: WADL / WSDL 2.0 / guia do dev / configs.  
- Capturar tráfego completo via **proxy** (Burp, mitmproxy, etc.) — **inclua headers, body e cookies**.

### 2) Analisar requisições coletadas
Procure por:
- Headers incomuns (ex.: `X-Auth-Token`, `X-Client-Id`).  
- Segmentos repetidos no path (datas, IDs).  
- Último elemento sem extensão → possível endpoint/ação.  
- Valores estruturados (JSON / XML / payloads customizados).

**Técnica rápida**: modificar segmento suspeito:
- Se retorna **404** → provavelmente path físico.  
- Se retorna erro de aplicação (4xx/5xx sem 404) → provavelmente parâmetro tratado pela app.

---

## 🎯 Otimizando o fuzzing
1. Classificar valores coletados: **válidos vs inválidos** — focar em margens (0, -1, strings, tamanhos extremos).  
2. Identificar limites e sequências (IDs, datas, offsets).  
3. Contextualizar fuzzing:
   - JSON → tipos errados, campos extras, omissões, nested inválido.  
   - Headers → fuzz em campos usados como parâmetros.  
   - Path params → encodings, traversal (`../`), tamanhos extremos.  
4. Emular autenticação: tokens, HMAC, nonces, timestamps. Se custom, capture e replique.

---

## ✅ Checklist rápido (copiar/colar)
- [ ] Obter documentação/contrato (WADL/WSDL/README)  
- [ ] Capturar tráfego completo via proxy (headers + body + params)  
- [ ] Identificar endpoints e métodos HTTP  
- [ ] Mapear parâmetros: path / query / body / headers  
- [ ] Identificar campos sensíveis (IDs, roles, balances)  
- [ ] Classificar parâmetros por tipo (int / string / date / enum)  
- [ ] Testar IDOR / manipulação de IDs / authorization bypass  
- [ ] Fuzzing direcionado (tipos, limites, caracteres especiais)  
- [ ] Testar autenticação/autorização (tokens, roles, scopes)  
- [ ] Testar rate limiting / anti-abuse (429)  
- [ ] Testar parsing de payloads (XXE, deserialização)  
- [ ] Verificar mensagens de erro (leaks / stack traces)  
- [ ] Registrar e reproduzir requisições que causaram falhas

---

## 📌 Exemplos práticos

**Detectar parâmetro no path**

Se o segmento varia muito → provavelmente parâmetro (ID).

**Headers como parâmetros**
```http
GET /api/resource HTTP/1.1
Host: api.exemplo
X-Client-Id: 12345
X-Auth-Token: abcdefg
```

Fuzzar `X-Client-Id` / `X-Auth-Token` pode revelar comportamentos.

## 🔬 Fuzzing JSON — checklist rápido

- Tipos errados (string ↔ number)
    
- Campos extras (`isAdmin: true`)
    
- Campos omitidos (validação ausente)
    
- Campos aninhados inválidos
    
- Valores muito longos / curtos (buffer / DoS)
    
- Confirmar validação **server-side**

## 🔐 Autenticação & sessão

- Tokens comuns: **JWT**, API keys, HMAC signatures.
    
- Testes importantes:
    
    - Replay de tokens
        
    - Tampering de JWT (alterar claims sem re-sign)
        
    - Tokens expirados / future tokens
        
    - Escalonamento de scope (scope escalation)
        
- Para schemes customizados: capture tráfego e reproduza algoritmo (quando possível).

## 🧭 Ordem prática sugerida

1. Recolher documentação.
    
2. Capturar tráfego completo.
    
3. Mapear endpoints & parâmetros.
    
4. Priorizar parâmetros de maior risco (IDs, roles).
    
5. Planejar fuzzing dirigido.
    
6. Emular autenticação e testar autorização.
    
7. Registrar evidências e repro steps.
    
8. Propor mitigações.

## 🛡️ Mitigações essenciais (resumo)

- Validação server-side (nunca confiar no cliente).
    
- DTOs / whitelists — evitar expor modelos internos.
    
- Autorização por objeto/ação.
    
- Validação de schema (JSON Schema).
    
- Mensagens de erro seguras (sem leaks).
    
- Rate-limiting e logs/auditoria.
    
- Tokens: assinatura, validade curta, refresh seguro.
    
- Uploads/parsing: proteção contra XXE e deserialização insegura.

## 📎 Referência rápida — códigos HTTP

- **200** OK — recurso retornado
- **201** Created — recurso criado
- **400** Bad Request — entrada inválida
- **401** Unauthorized — autenticação requerida / inválida
- **403** Forbidden — não autorizado
- **404** Not Found — recurso/path inexistente
- **429** Too Many Requests — rate limiting
- **5xx** Erro servidor — investigar leak / tratamento

## 🧪 Exemplos de uso / testes — Node.js e Java

### Exemplo Node.js — requisição com headers, body JSON e manipulação de token (axios)

```node
// npm install axios
const axios = require('axios');

async function callApi() {
  const url = 'https://api.exemplo/v1/resource/1234';
  const token = 'eyJ...'; // token capturado (JWT / custom)
  try {
    const resp = await axios({
      method: 'post',
      url,
      headers: {
        'Authorization': `Bearer ${token}`,
        'X-Client-Id': '12345',
        'Content-Type': 'application/json'
      },
      data: {
        action: 'update',
        amount: 100,
        // tente injetar campos para testar mass assignment:
        // isAdmin: true
      },
      timeout: 5000
    });
    console.log('status', resp.status);
    console.log('body', resp.data);
  } catch (err) {
    if (err.response) {
      console.log('erro status', err.response.status);
      console.log('erro body', err.response.data);
    } else {
      console.log('erro', err.message);
    }
  }
}

callApi();

```

- Use esse script para reproduzir requisições capturadas e testar variações de headers/body.
- Experimente alterar `X-Client-Id`, `Authorization`, tipos de campos e tamanhos (payload grande).

Exemplo Node.js — fuzzing simples de JSON (payloads com tipos e valores)

```node
// Exemplo simples para gerar variações de payload
const axios = require('axios');

const payloads = [
  { amount: "100" },        // tipo errado
  { amount: -1 },           // valor inválido
  { amount: 9999999999 },   // valor extremo
  { isAdmin: true },        // campo extra
  { nested: { a: "x".repeat(10000) } } // valor gigante (DoS test)
];

async function fuzz() {
  for (const p of payloads) {
    try {
      const r = await axios.post('https://api.exemplo/v1/resource/1234', p, {
        headers: { 'Content-Type': 'application/json', 'Authorization': 'Bearer eyJ...' },
        validateStatus: () => true
      });
      console.log('payload', p, '=>', r.status);
    } catch (e) {
      console.error('erro', e.message);
    }
  }
}
fuzz();

```

Exemplo Java — HttpClient (Java 11+) com header custom e JSON

```java
// Requer Java 11+
import java.net.URI;
import java.net.http.*;
import java.net.http.HttpRequest.BodyPublishers;

public class SimpleCall {
  public static void main(String[] args) throws Exception {
    HttpClient client = HttpClient.newHttpClient();
    String url = "https://api.exemplo/v1/resource/1234";
    String json = "{\"action\":\"update\",\"amount\":100}";

    HttpRequest req = HttpRequest.newBuilder()
      .uri(URI.create(url))
      .header("Content-Type", "application/json")
      .header("X-Client-Id", "12345")
      .header("Authorization", "Bearer eyJ...")
      .POST(BodyPublishers.ofString(json))
      .build();

    HttpResponse<String> resp = client.send(req, HttpResponse.BodyHandlers.ofString());
    System.out.println("Status: " + resp.statusCode());
    System.out.println("Body: " + resp.body());
  }
}

```

Exemplo Java — Spring (RestTemplate) para testar endpoints e headers

```java
// Spring Boot + RestTemplate
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;

public class SpringClient {
  public static void main(String[] args) {
    RestTemplate rt = new RestTemplate();
    String url = "https://api.exemplo/v1/resource/1234";

    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.APPLICATION_JSON);
    headers.set("X-Client-Id", "12345");
    headers.setBearerAuth("eyJ...");

    String body = "{\"action\":\"update\",\"amount\":100}";
    HttpEntity<String> req = new HttpEntity<>(body, headers);

    ResponseEntity<String> resp = rt.exchange(url, HttpMethod.POST, req, String.class);
    System.out.println("Status: " + resp.getStatusCodeValue());
    System.out.println("Body: " + resp.getBody());
  }
}

```
- Use para reproduzir requisições e observar respostas/erros do servidor.
- Capture e compare cabeçalhos e corpos.

## 📝 Dicas finais rápidas

- Sempre capture requisições reais via proxy e reproduza-as com scripts (Node/Java) — facilita fuzzing e report.
- Logue respostas completas (status + body) para identificar diferenças sutis.
- Teste autenticação em profundidade: replay, tamper, expiração, scope.
- Priorize testes em parâmetros que representam IDs e flags (alto impacto: IDOR, mass-assignment).
