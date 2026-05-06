---
tags: [security, owasp, api7, ssrf, server-side-request-forgery, cloud, metadata, pentest]
aliases:
  - API7 Server Side Request Forgery
  - Server Side Request Forgery
  - SSRF
  - Falsificação de Requisição do Lado do Servidor
---

# 🌐 API7:2023 — Server Side Request Forgery (SSRF)

**Resumo:**  
**Server Side Request Forgery (SSRF)** ocorre quando uma API aceita uma URL, host ou destino controlado pelo usuário e faz uma requisição do lado do servidor sem validação adequada.

O impacto é perigoso porque o servidor pode acessar recursos que o atacante não acessaria diretamente, como redes internas, serviços administrativos, metadados de cloud, painéis internos e endpoints protegidos por firewall.

---

## 🔗 Associações no Obsidian

- Hub: [[API Security]]
- Testes de API: [[Guia de consulta rápida — Avaliação REST]], [[Pentest]], [[Vulnerabilidades]]
- Abuso e limites: [[Unrestricted Resource Consumption|API4:2023 — Rate Limiting]], [[unrestricted-access-to-sensitive-business-flows|API6:2023 — Sensitive Business Flows]]
- Segredos e cloud: [[Vazamentos e Tokens de API]], [[security-misconfiguration|API8:2023 — Security Misconfiguration]], [[GHAS]], [[Containers e Orquestração]]
- Resposta e investigação: [[Segurança de Aplicativos - Resposta a Incidentes]]

---

## 🧠 Descrição da Vulnerabilidade

SSRF aparece quando a aplicação faz requisições externas baseadas em entrada do usuário.

Exemplos comuns:

- Importar imagem por URL.
- Buscar avatar remoto.
- Validar webhook informado pelo cliente.
- Gerar preview de link.
- Converter HTML/PDF a partir de uma URL.
- Integrar com callbacks configuráveis.
- Baixar arquivos de armazenamento remoto.
- Consultar feeds, XML, JSON ou endpoints informados pelo usuário.

Se a API não controla o destino, o atacante pode fazer o servidor acessar alvos internos ou sensíveis.

---

## ⚠️ A API é Vulnerável?

A API pode estar vulnerável se:

- Aceita URLs completas fornecidas pelo usuário.
- Permite informar `host`, `port`, `scheme` ou `callbackUrl`.
- Faz fetch de recursos remotos sem allowlist.
- Bloqueia apenas strings simples, como `localhost`, mas não resolve DNS corretamente.
- Permite redirects para destinos internos.
- Aceita protocolos perigosos ou desnecessários.
- Não restringe acesso a IPs privados, loopback, link-local ou metadados cloud.
- O servidor possui acesso a redes internas que o usuário externo não possui.

Destinos perigosos comuns:

- `http://127.0.0.1`
- `http://localhost`
- `http://169.254.169.254`
- `http://metadata.google.internal`
- `http://[::1]`
- Serviços internos como `http://admin.internal`, `http://redis:6379`, `http://kubernetes.default`

---

## 💣 Cenários de Ataque

### 🧨 Cenário #1 — Importação de imagem por URL

Endpoint legítimo:

```http
POST /api/profile/avatar
{
  "imageUrl": "https://example.com/avatar.png"
}
```

O atacante altera para:

```http
POST /api/profile/avatar
{
  "imageUrl": "http://127.0.0.1:8080/admin"
}
```

➡️ **Problema:** o back-end busca qualquer URL informada.

➡️ **Resultado:** o servidor acessa um endpoint interno em nome do atacante.

### 🧨 Cenário #2 — Acesso a metadados de cloud

Payload:

```http
POST /api/import
{
  "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/"
}
```

➡️ **Problema:** a API permite acesso a endereços link-local.

➡️ **Resultado:** possível exposição de credenciais temporárias da instância cloud.

### 🧨 Cenário #3 — Webhook com callback malicioso

Cadastro de webhook:

```http
POST /api/webhooks
{
  "callbackUrl": "https://client.example/webhook"
}
```

O atacante usa:

```http
POST /api/webhooks
{
  "callbackUrl": "http://internal-service.local/debug"
}
```

➡️ **Problema:** a API dispara chamadas para destinos internos.

➡️ **Resultado:** varredura, interação ou exploração de serviços internos.

### 🧨 Cenário #4 — Redirect para rede interna

A aplicação bloqueia `localhost`, mas aceita:

```http
https://attacker.example/redirect
```

Esse endpoint responde com:

```http
302 Location: http://127.0.0.1:8080/admin
```

➡️ **Problema:** a validação ocorre apenas antes do redirect.

➡️ **Resultado:** bypass da validação inicial e acesso a recurso interno.

---

## 💻 Exemplos de Código — Vulnerável e Mitigado

### ❌ Node.js (Express) — Exemplo Vulnerável

```javascript
import axios from 'axios';

router.post('/preview', async (req, res) => {
  const { url } = req.body;
  const response = await axios.get(url);

  res.json({ preview: response.data.slice(0, 500) });
});
```

**Problemas:**

- Aceita qualquer URL.
- Permite acesso a redes internas.
- Segue redirects sem revalidar destino.
- Não controla tempo, tamanho de resposta ou protocolos.

### ✅ Node.js — Mitigação Recomendada

```javascript
import axios from 'axios';
import dns from 'node:dns/promises';
import net from 'node:net';

const allowedHosts = new Set(['example.com', 'cdn.example.com']);

function isPrivateIp(ip) {
  return (
    ip.startsWith('10.') ||
    ip.startsWith('127.') ||
    ip.startsWith('169.254.') ||
    ip.startsWith('172.16.') ||
    ip.startsWith('192.168.') ||
    ip === '::1'
  );
}

async function validateUrl(rawUrl) {
  const parsed = new URL(rawUrl);

  if (!['https:'].includes(parsed.protocol)) {
    throw new Error('Protocolo não permitido');
  }

  if (!allowedHosts.has(parsed.hostname)) {
    throw new Error('Host não permitido');
  }

  const addresses = await dns.lookup(parsed.hostname, { all: true });
  if (addresses.some(({ address }) => net.isIP(address) && isPrivateIp(address))) {
    throw new Error('Destino privado bloqueado');
  }

  return parsed.toString();
}

router.post('/preview', async (req, res) => {
  const safeUrl = await validateUrl(req.body.url);

  const response = await axios.get(safeUrl, {
    maxRedirects: 0,
    timeout: 3000,
    maxContentLength: 1024 * 1024,
  });

  res.json({ preview: String(response.data).slice(0, 500) });
});
```

**Mitigação aplicada:**

- Allowlist de hosts.
- Apenas HTTPS.
- Bloqueio de IPs privados/link-local.
- Redirects desabilitados.
- Timeout e limite de tamanho.

### ❌ Java (Spring Boot) — Exemplo Vulnerável

```java
@PostMapping("/import")
public String importUrl(@RequestBody ImportRequest req) {
    return restTemplate.getForObject(req.getUrl(), String.class);
}
```

**Problemas:**

- A URL vem diretamente do usuário.
- O servidor pode acessar recursos internos.
- Não há validação de host, protocolo ou IP resolvido.

### ✅ Java — Mitigação Recomendada

```java
private static final Set<String> ALLOWED_HOSTS = Set.of(
    "example.com",
    "cdn.example.com"
);

private void validateUrl(String rawUrl) throws Exception {
    URI uri = new URI(rawUrl);

    if (!"https".equalsIgnoreCase(uri.getScheme())) {
        throw new IllegalArgumentException("Protocolo não permitido");
    }

    if (!ALLOWED_HOSTS.contains(uri.getHost())) {
        throw new IllegalArgumentException("Host não permitido");
    }

    InetAddress address = InetAddress.getByName(uri.getHost());
    if (address.isAnyLocalAddress()
            || address.isLoopbackAddress()
            || address.isLinkLocalAddress()
            || address.isSiteLocalAddress()) {
        throw new IllegalArgumentException("Destino privado bloqueado");
    }
}

@PostMapping("/import")
public ResponseEntity<?> importUrl(@RequestBody ImportRequest req) throws Exception {
    validateUrl(req.getUrl());

    String body = restTemplate.getForObject(req.getUrl(), String.class);
    return ResponseEntity.ok(body);
}
```

**Mitigação aplicada:**

- Validação de protocolo.
- Allowlist de hosts.
- Bloqueio de endereços internos.
- Regras aplicadas antes da requisição server-side.

---

## 🛡️ Como Prevenir

- [ ] Evitar aceitar URLs arbitrárias do usuário.
- [ ] Usar allowlist explícita de domínios e serviços permitidos.
- [ ] Permitir apenas protocolos necessários, normalmente `https`.
- [ ] Resolver DNS e bloquear IPs privados, loopback, link-local e metadados cloud.
- [ ] Revalidar o destino após redirects ou desabilitar redirects.
- [ ] Aplicar timeout, limite de tamanho e limite de conexões.
- [ ] Isolar serviços que fazem fetch externo em rede com egress restrito.
- [ ] Bloquear acesso a metadados cloud quando não for necessário.
- [ ] Monitorar chamadas server-side incomuns.
- [ ] Registrar destino, usuário, origem e resultado das requisições externas.

---

## 🧱 Boas Práticas

- Não confiar em validações baseadas apenas em regex.
- Normalizar e parsear URLs com bibliotecas confiáveis.
- Validar o host após resolução DNS, não só o texto informado.
- Cuidado com DNS rebinding.
- Tratar redirects como novo destino a ser validado.
- Separar integrações externas legítimas de fetch arbitrário.
- Usar proxy de saída com políticas de egress quando possível.
- Em cloud, proteger endpoints de metadata e limitar permissões IAM.

---

## ✅ Checklist de Segurança — SSRF

- [ ] Nenhum endpoint aceita URL arbitrária sem validação.
- [ ] Existe allowlist de destinos permitidos.
- [ ] Protocolos não necessários são bloqueados.
- [ ] IPs internos, loopback, link-local e metadata são bloqueados.
- [ ] Redirects são bloqueados ou revalidados.
- [ ] Timeouts e limites de tamanho estão configurados.
- [ ] O serviço possui egress controlado.
- [ ] Credenciais cloud seguem menor privilégio.
- [ ] Tentativas suspeitas são logadas e monitoradas.
- [ ] Testes cobrem `localhost`, `127.0.0.1`, `169.254.169.254`, IPv6 e redirects.

---

## 🔥 Resumo

> SSRF transforma o servidor em um cliente controlado pelo atacante.

Se uma API faz requisições para destinos definidos pelo usuário sem validação forte, o atacante pode alcançar redes internas, metadados de cloud e serviços que deveriam estar invisíveis externamente.
