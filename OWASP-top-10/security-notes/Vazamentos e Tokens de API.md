
---
tags: [apisec, api-security, secrets, tokens, devsecops, bug-bounty]
aliases:
  - Vazamento de segredos
  - Tokens de API
  - API Secrets
---

# 🔍 Ferramentas de Segurança — Detecção de Segredos e Tokens

Um conjunto de ferramentas voltadas à **análise de vazamento de segredos**, **validação de tokens** e **varredura de credenciais expostas** em aplicações e repositórios.

---

## 🔗 Associações no Obsidian

- Hub: [[API Security]]
- Relacionado a autenticação: [[broken-authentication|API2:2023 — Autenticação Quebrada]], [[JWT checklist]], [[Checklist ameaças OAuth 2.0]]
- Relacionado a revisão e exposição: [[GHAS]], [[github]], [[security-misconfiguration|Security Misconfiguration]], [[Guia de consulta rápida — Avaliação REST]]
- Relacionado a abuso de API: [[Unrestricted Resource Consumption|API4:2023 — Unrestricted Resource Consumption]], [[server-side-request-forgery|API7:2023 — SSRF]]

---

## 🧰 Principais Ferramentas

### 1. [aquasecurity/trivy](https://github.com/aquasecurity/trivy)
Scanner de vulnerabilidades e **configurações incorretas de propósito geral**, que também **detecta chaves e segredos de API**.  
🧩 Ideal para pipelines DevSecOps e auditorias de container.

---

### 2. [blacklanternsecurity/badsecrets](https://github.com/blacklanternsecurity/badsecrets)
Biblioteca que realiza a **detecção de segredos conhecidos ou vulneráveis** em diferentes plataformas e linguagens.  
🔎 Útil para análise de código-fonte e revisão de pacotes.

---

### 3. [d0ge/sign-saboteur](https://github.com/d0ge/sign-saboteur)
Extensão do **Burp Suite** que permite **editar, assinar e verificar tokens web assinados** (JWT, JWS, etc.).  
💥 Excelente para exploração e testes de integridade de tokens.

---

### 4. [mazen160/secrets-patterns-db](https://github.com/mazen160/secrets-patterns-db)
O **maior banco de dados open-source** com **padrões de detecção de segredos** — incluindo **chaves, tokens, senhas e credenciais**.  
🧠 Recomendado para integrar em pipelines de SAST ou ferramentas customizadas.

---

### 5. [momenbasel/KeyFinder](https://github.com/momenbasel/KeyFinder)
Ferramenta leve que permite **encontrar chaves sensíveis enquanto você navega na web**.  
🌐 Excelente para **bug bounty** e **pentests web dinâmicos**.

---

### 6. [streaak/keyhacks](https://github.com/streaak/keyhacks)
Repositório prático com **técnicas para verificar rapidamente a validade de chaves de API** encontradas em vazamentos.  
⚙️ Perfeito para **bug hunters** e **auditoria manual de segredos**.

---

### 7. [trufflesecurity/truffleHog](https://github.com/trufflesecurity/truffleHog)
Ferramenta popular que **encontra credenciais expostas** em **repositórios Git, histórico de commits e arquivos**.  
🐷 Um dos **scanners de segredos mais utilizados** no mercado.

---

### 8. [projectdiscovery/nuclei-templates](https://github.com/projectdiscovery/nuclei-templates)
Conjunto de **templates para o Nuclei**, permitindo testar **tokens de API** em **vários endpoints de serviços**.  
🧠 Excelente complemento para pipelines automatizados e testes direcionados.

```shell
nuclei -t token-spray/ -var token=token_list.txt
```

---

### 9. [Key-Checker](https://github.com/daffainfo/Key-Checker)   
🔑 Scripts em Go para verificar a **validade de chaves de API e tokens de acesso**.  
Ideal para **análises de impacto de vazamento**, **bug bounty** e **verificação de segredos** em **pipelines CI/CD**.

---

### 10. [Mantra](https://github.com/brosck/mantra?utm_source=chatgpt.com) 
🕵️‍♂️Ferramenta escrita em Go que busca vazamentos de **chaves de API** em arquivos JavaScript e páginas HTML. Ideal para auditorias rápidas de front-end, bug bounty e pipelines de detecção de segredos.

---

## 📘 Dica Extra

> 💡 Combine ferramentas como **Trivy** e **TruffleHog** para uma abordagem híbrida:  
> - **Trivy**: análise de imagens, pacotes e IaC.  
> - **TruffleHog**: varredura profunda por segredos expostos no histórico de código.

---

## Causas comuns de vazamentos

- **Registros e informações de depuração** : Chaves e tokens podem ser registrados ou impressos inadvertidamente durante os processos de depuração.

- **Arquivos de configuração** : Incluindo chaves e tokens em arquivos de configuração publicamente acessíveis (por exemplo, arquivos .env, config.json, settings.py ou .aws/credentials).

- **Inserção direta no código-fonte** : Os desenvolvedores podem, sem intenção, deixar chaves de API ou tokens diretamente no código-fonte.

```python
# Example of hardcoded API key
api_key = "1234567890abcdef"
```

- **Repositórios públicos** : O envio acidental de chaves e tokens confidenciais para sistemas de controle de versão publicamente acessíveis, como o GitHub.

### Scan de organização GitHub com TruffleHog

```powershell
# Scan a Github Organization
docker run --rm -it -v "$PWD:/pwd" trufflesecurity/trufflehog:latest github --org=trufflesecurity
```

|**Elemento**|**Descrição**|**Função / Observação**|
|---|---|---|
|`docker run`|Executa um container Docker baseado em uma imagem.|Inicia o container que executará o TruffleHog.|
|`--rm`|Remove automaticamente o container ao final da execução.|Evita acúmulo de containers inativos no sistema.|
|`-it`|Combina `-i` (modo interativo) e `-t` (pseudo-terminal).|Permite interação e exibição direta dos resultados no terminal.|
|`-v "$PWD:/pwd"`|Cria um _bind mount_ do diretório atual (`$PWD`) para o diretório `/pwd` dentro do container.|Facilita salvar relatórios ou logs localmente.|
|`trufflesecurity/trufflehog:latest`|Imagem Docker oficial do TruffleHog (última versão estável).|Contém a ferramenta usada para escanear repositórios, imagens e serviços.|
|`github`|Subcomando do TruffleHog.|Indica que o alvo do scan será o **GitHub** (repos, orgs ou usuários).|
|`--org=trufflesecurity`|Define a **organização GitHub** a ser analisada.|O TruffleHog varrerá **todos os repositórios públicos** da organização `trufflesecurity` em busca de segredos.|

### Scan de repositório GitHub, issues e pull requests

```powershell
docker run --rm -it -v "$PWD:/pwd" trufflesecurity/trufflehog:latest github --repo https://github.com/trufflesecurity/test_keys --issue-comments --pr-comments
```

| **Elemento**                                          | **Descrição**                                                                   | **Função / Observação**                                                        |
| ----------------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `docker run`                                          | Executa um container Docker com base em uma imagem.                             | Inicia o container que rodará o TruffleHog.                                    |
| `--rm`                                                | Remove automaticamente o container após sua execução.                           | Mantém o ambiente limpo, sem containers residuais.                             |
| `-it`                                                 | Combina `-i` (interativo) e `-t` (pseudo-TTY).                                  | Permite acompanhar logs e encerrar manualmente.                                |
| `-v "$PWD:/pwd"`                                      | Cria um bind mount do diretório atual (`$PWD`) para `/pwd` dentro do container. | Permite que os resultados do scan sejam salvos localmente.                     |
| `trufflesecurity/trufflehog:latest`                   | Imagem Docker oficial do TruffleHog.                                            | Contém a ferramenta usada para escanear repositórios, imagens e sistemas.      |
| `github`                                              | Subcomando do TruffleHog.                                                       | Indica que o alvo do scan será um **repositório GitHub**.                      |
| `--repo https://github.com/trufflesecurity/test_keys` | URL do repositório a ser analisado.                                             | Define o destino da varredura em busca de segredos.                            |
| `--issue-comments`                                    | Habilita análise de comentários em _issues_ do GitHub.                          | Amplia a detecção de segredos para fora do código-fonte.                       |
| `--pr-comments`                                       | Habilita análise de comentários em _pull requests_.                             | Verifica se chaves, tokens ou credenciais foram expostos em discussões de PRs. |

- **Inserção direta de chaves e credenciais em imagens Docker** : as chaves de API e as credenciais podem estar inseridas diretamente no código em imagens Docker hospedadas no DockerHub ou em registros privados.

### Scan de imagem Docker com TruffleHog

```powershell
# Scan a Docker image for verified secrets
Scan a Docker image for verified secrets
docker run --rm -it -v "$PWD:/pwd" trufflesecurity/trufflehog:latest docker --image trufflesecurity/secrets
```

## Desconstruindo o comando

|Elemento|Função|Dica|
|---|---|---|
|`docker run`|Executa o container|Comando base|
|`--rm`|Remove o container após execução|Evita resíduos|
|`-it`|Interativo + TTY|Permite visualização em tempo real|
|`-v "$PWD:/pwd"`|Monta o diretório local|Ideal para salvar relatórios|
|`trufflesecurity/trufflehog:latest`|Imagem Docker utilizada|Scanner de segredos|
|`docker --image trufflesecurity/secrets`|Parâmetros do TruffleHog|Define o alvo da varredura|

---

## Bancos de padrões e validação de tokens

Caso precise de ajuda para identificar o serviço que gerou o token, você pode consultar o banco de dados [mazen160/secrets-patterns-db](https://github.com/mazen160/secrets-patterns-db) . Trata-se do maior banco de dados de código aberto para detecção de segredos, chaves de API, senhas, tokens e muito mais. Este banco de dados contém padrões de expressões regulares para diversos segredos.

```json
patterns:
  - pattern:
      name: AWS API Gateway
      regex: '[0-9a-z]+.execute-api.[0-9a-z._-]+.amazonaws.com'
      confidence: low
  - pattern:
      name: AWS API Key
      regex: AKIA[0-9A-Z]{16}
      confidence: high
```

Use [o streaak/keyhacks](https://github.com/streaak/keyhacks) ou leia a documentação do serviço para encontrar uma maneira rápida de verificar a validade de uma chave de API.

- **Exemplo** : Token da API do bot do Telegram

```powershell
curl https://api.telegram.org/bot<TOKEN>/getMe
```
