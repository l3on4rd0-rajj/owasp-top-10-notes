
---
tags: [security, owasp, api4, rate-limiting, dos, nodejs, java]
aliases:
  - API4 Unrestricted Resource Consumption
  - Falta de Limitação de Recursos
  - Rate Limiting em APIs
---

# 🚧 API4:2023 — Falta de Limitação de Recursos e Rate Limiting (DoS / Abuse)

## 🔗 Associações no Obsidian

- Hub: [[API Security]]
- Autenticação e abuso: [[broken-authentication|API2:2023 — Autenticação Quebrada]], [[unrestricted-access-to-sensitive-business-flows|API6:2023 — Sensitive Business Flows]], [[Checklist ameaças OAuth 2.0]]
- Segurança de API: [[Guia de consulta rápida — Avaliação REST]], [[server-side-request-forgery|API7:2023 — SSRF]], [[Vazamentos e Tokens de API]]
- Infraestrutura e operação: [[docker compose]], [[minikube]]

---

## 🎯 Agentes de Ameaça e Vetores de Ataque

| Agentes / Vetores                                                                                                                                                         | Vulnerabilidade                                                                                                                           | Impactos                                                                                                                        |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| API específica: Média de explorabilidade                                                                                                                                  | Prevalência generalizada: fácil detecção                                                                                                  | Nível técnico grave: específico para o negócio                                                                                  |
| A exploração requer requisições simples. Ferramentas automatizadas podem gerar grandes volumes de tráfego (DoS), causando sobrecarga de CPU, memória e custo operacional. | É comum encontrar APIs sem limitação de interação. Parâmetros que controlam volume de resposta ou operações em lote podem revelar falhas. | Pode causar **negação de serviço (DoS)** e aumento de custos de infraestrutura e APIs terceirizadas (ex: SMS, storage, e-mail). |

---

## ❓ A API é vulnerável?

Uma API é vulnerável se **não possuir limites definidos** ou configurados incorretamente em recursos como:

- Tempo máximo de execução  
- Memória máxima alocável  
- Número máximo de descritores de arquivo/processos  
- Tamanho máximo de upload  
- Limite de registros por página  
- Operações em lote (ex.: GraphQL)  
- Limite de gastos de provedores terceirizados  

---

## 💥 Cenários de Ataque

### 🧪 Cenário #1 — SMS de Recuperação de Senha

```http
POST /initiate_forgot_password
{
  "step": 1,
  "user_number": "6501113434"
}

```

➡️ A API chama um serviço externo de SMS que cobra por mensagem.  
Um atacante executa milhares de requisições, gerando prejuízo financeiro.

### 🧪 Cenário #2 — GraphQL sem Limite de Lote

```node
POST /graphql
[
  {"query": "mutation {uploadPic(name: \"pic1\", base64_pic: \"R0FOIEFOR0xJVA…\") {url}}"},
  {"query": "mutation {uploadPic(name: \"pic2\", base64_pic: \"R0FOIEFOR0xJVA…\") {url}}"},
  ...
]
```

➡️ O atacante envia centenas de mutações em lote, consumindo toda a memória do servidor e causando **DoS**.

### 🧪 Cenário #3 — Cache e Custos em Armazenamento

Um provedor de arquivos usa cache até 15 GB.  
Após upload de um arquivo de 18 GB, todos os clientes começam a baixar a nova versão simultaneamente.  

➡️ Fatura da nuvem sobe de **US$ 13 para US$ 8.000** em um mês.

---

## 🧰 Checklist — Como Prevenir

- [ ] Definir limites de **memória**, **CPU**, **processos** e **tempo de execução**.
- [ ] Impor **tamanho máximo** para parâmetros, strings e uploads.
- [ ] Implementar **Rate Limiting / Throttling** (limite por tempo e operação).
- [ ] Controlar **quantas vezes** um usuário pode executar uma operação (OTP, reset de senha, etc.).
- [ ] Validar **parâmetros de paginação** e de resposta (número máximo de registros).
- [ ] Configurar **limites de gastos** e **alertas de faturamento** em APIs de terceiros.

---

## 💻 Exemplos de Implementação

### Node.js (Express + Rate Limiting)

```node.js
// rateLimit.js
import rateLimit from "express-rate-limit";

const apiLimiter = rateLimit({
  windowMs: 1 * 60 * 1000, // 1 minuto
  max: 10, // máximo 10 requisições por IP
  message: { error: "Muitas requisições, tente novamente mais tarde." },
});

export default apiLimiter;
```

```node.js
// 
import express from "express";
import apiLimiter from "./rateLimit.js";

const app = express();
app.use(express.json());

// Aplicando limitação apenas em endpoints críticos
app.use("/api/auth/forgot-password", apiLimiter);

app.post("/api/auth/forgot-password", (req, res) => {
  const { user_number } = req.body;
  // Lógica de envio do SMS
  res.json({ message: `Código enviado para ${user_number}` });
});

app.listen(3000, () => console.log("Servidor rodando na porta 3000"));
```

### Java (Spring Boot + Bucket4j)

```java
// Importações principais
import io.github.bucket4j.*;                // Biblioteca Bucket4j (implementa rate limiting com token bucket)
import jakarta.servlet.*;                  // Interfaces para criar filtros servlet
import jakarta.servlet.http.*;             // Permite manipular requisições e respostas HTTP
import org.springframework.stereotype.Component; // Marca o filtro como componente Spring
import java.io.IOException;
import java.time.Duration;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class RateLimitFilter implements Filter {

    // Armazena os "baldes" de requisições, um por IP
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    /**
     * Cria um novo "bucket" de requisições para um cliente (ex: um IP específico).
     * 
     * - Refill.intervally(10, Duration.ofMinutes(1)): permite 10 novas requisições por minuto
     * - Bandwidth.classic(10, refill): define um limite de 10 tokens no total
     * - Cada requisição consome 1 token
     */
    private Bucket createNewBucket() {
        Refill refill = Refill.intervally(10, Duration.ofMinutes(1)); // Reabastece 10 tokens a cada 1 minuto
        Bandwidth limit = Bandwidth.classic(10, refill);              // Capacidade máxima do bucket
        return Bucket.builder().addLimit(limit).build();
    }

    /**
     * Intercepta todas as requisições HTTP antes de chegarem ao controlador.
     * 
     * - Verifica se o IP possui um bucket de requisições.
     * - Tenta consumir 1 token (representa 1 requisição).
     * - Se não houver token disponível → responde com HTTP 429 (Too Many Requests).
     */
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        
        HttpServletRequest req = (HttpServletRequest) request;
        String ip = req.getRemoteAddr(); // Identifica o cliente pela origem IP
        
        // Busca o bucket correspondente ou cria um novo
        Bucket bucket = buckets.computeIfAbsent(ip, k -> createNewBucket());

        if (bucket.tryConsume(1)) {
            // Se ainda há tokens disponíveis, segue o fluxo normal da requisição
            chain.doFilter(request, response);
        } else {
            // Se excedeu o limite → retorna 429 (Too Many Requests)
            ((HttpServletResponse) response).setStatus(429);
            response.getWriter().write("Muitas requisições. Tente novamente mais tarde.");
        }
    }
}
```
