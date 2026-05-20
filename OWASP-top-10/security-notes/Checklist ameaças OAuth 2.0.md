---
title: 🧩 Validação de URI de Redirecionamento — OAuth 2.0
tags: [oauth2, redirect-uri, authorization-code, implicit-grant, security, appsec]
---

## 🔗 Associações no Obsidian

- Hub: [[API Security]]
- Autenticação e tokens: [[broken-authentication|Broken Authentication]], [[JWT checklist]], [[Vazamentos e Tokens de API]]
- Testes REST: [[Guia de consulta rápida — Avaliação REST]], [[Pentest]]
- Resposta e controles: [[Unrestricted Resource Consumption|Rate Limiting]], [[Segurança de Aplicativos - Resposta a Incidentes]]

---

#### **LISTA DE VERIFICAÇÃO**

- [ ] **Ameaça:** Validação insuficiente do URI de redirecionamento

---

### [Ataques de Validação de URI de Redirecionamento em Concessão de Código de Autorização](https://web.archive.org/web/20201101013657/https://tools.ietf.org/id/draft-ietf-oauth-security-topics-15.html#name-redirect-uri-validation-att)

Para um cliente que utiliza o tipo de concessão **Authorization Code**, um ataque pode ocorrer da seguinte forma:

Suponha que o padrão de URL de redirecionamento `https://*.somesite.example/*` esteja registrado para o cliente com o ID `s6BhdRkqt3`.  
A intenção é permitir que qualquer subdomínio de `somesite.example` seja um URI de redirecionamento válido, por exemplo:  
`https://app1.somesite.example/redirect`.

Uma implementação ingênua no servidor de autorização pode interpretar o curinga `*` como “qualquer caractere”, e não apenas como “qualquer subdomínio válido”.  
Assim, o servidor de autorização poderia aceitar `https://attacker.example/.somesite.example` — um domínio controlado por um atacante.

---

#### 🧠 **Cenário do ataque**

1. O atacante engana o usuário para abrir um URL adulterado, apontando para `https://www.evil.example`.
2. Esse URL inicia a seguinte solicitação de autorização:

```http
GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=9ad67f13
     &redirect_uri=https%3A%2F%2Fattacker.example%2F.somesite.example
     HTTP/1.1
Host: server.somesite.example
```

- [ ] Ameaça: Vazamento de credenciais por meio de cabeçalhos de referência

### [Vazamento de credenciais por meio de cabeçalhos de referência](https://web.archive.org/web/20201101013657/https://tools.ietf.org/id/draft-ietf-oauth-security-topics-15.html#name-credential-leakage-via-refe)

O conteúdo do URI de solicitação de autorização ou do URI de resposta de autorização pode ser divulgado involuntariamente a atacantes por meio do cabeçalho HTTP Referer vazando do site do AS ou do site do cliente, respectivamente. Mais importante ainda, códigos ou `state`valores de autorização podem ser divulgados dessa forma. 

#### [Vazamento do cliente OAuth](https://web.archive.org/web/20201101013657/https://tools.ietf.org/id/draft-ietf-oauth-security-topics-15.html#name-leakage-from-the-oauth-clie)

O vazamento do cliente OAuth exige que o cliente, como resultado de uma solicitação de autorização bem-sucedida, renderize uma página que 

- contém links para outras páginas sob o controle do atacante e um usuário clica em tal link, ou 
    
-  terceiros (anúncios em iframes, imagens, etc.), por exemplo, se a página contiver conteúdo gerado pelo usuário (blog).
    

Assim que o navegador acessa a página do atacante ou carrega o conteúdo de terceiros, o atacante recebe a URL de resposta de autorização e pode extrair `code`informações confidenciais `state`(e potencialmente dados de `access token` ).

#### [Consequências](https://web.archive.org/web/20201101013657/https://tools.ietf.org/id/draft-ietf-oauth-security-topics-15.html#name-consequences)

 Se o atacante obtiver o código ou token de acesso válido `state`, a proteção contra CSRF obtida pelo uso do cabeçalho Referer será perdida, resultando em ataques CSRF.

#### [Contramedidas](https://web.archive.org/web/20201101013657/https://tools.ietf.org/id/draft-ietf-oauth-security-topics-15.html#name-countermeasures-2)

A página gerada como resultado da resposta de autorização OAuth e o endpoint de autorização NÃO DEVEM incluir recursos de terceiros ou links para sites externos 

As seguintes medidas reduzem ainda mais as chances de um ataque bem-sucedido:

- Suprima o cabeçalho Referer aplicando uma política Referrer apropriada ao documento (seja como parte do meta atributo "referrer" ou definindo um cabeçalho Referrer-Policy). Por exemplo, o cabeçalho `Referrer-Policy: no-referrer`na resposta suprime completamente o cabeçalho Referer em todas as solicitações originadas do documento resultante 
    
- Use o código de autorização em vez dos tipos de resposta que causam a emissão de tokens de acesso a partir do endpoint de autorização. 
    
- Vincule o código de autorização a um cliente confidencial ou a um desafio PKCE. Nesse caso, o atacante não possui o segredo para solicitar a troca de código 
    
- Os códigos de autorização DEVEM ser invalidados pelo AS após o primeiro uso no endpoint do  . Por exemplo, se um AS invalidasse o código depois que o cliente legítimo o resgatasse, o atacante falharia ao tentar trocar esse código posteriormente.
    
    Isso não atenua o ataque se o atacante conseguir trocar o código por um token antes que o cliente legítimo o faça. Portanto, recomenda-se ainda que, quando houver uma tentativa de resgatar um código duas vezes, o AS DEVE revogar todos os tokens emitidos anteriormente com base nesse código.

- O `state`valor DEVE ser invalidado pelo cliente após seu primeiro uso no ponto de redirecionamento. Se isso for implementado e um atacante receber um token por meio do cabeçalho Referer do site do cliente, o token `state`já terá sido usado, invalidado pelo cliente e não poderá ser usado novamente pelo atacante. (Isso não ajuda se o token `state`vazar do site do AS, pois, nesse caso, o token `state` ainda não terá sido usado no ponto de redirecionamento do cliente. 
    


 - [ ] Ameaça: Vazamento de credenciais através do histórico do navegador

Quando um navegador navega para um endereço `client.example/redirection_endpoint?code=abcd`como resultado de um redirecionamento do endpoint de autorização de um provedor, a URL que inclui o código de autorização pode acabar no histórico do navegador. Um invasor com acesso ao dispositivo poderia obter o código e tentar reproduzi-lo.

- Use o modo de resposta POST do formulário em vez de um redirecionamento para a resposta de autorização.

- [ ] Ameaça: Vazamento de credenciais através do histórico do navegador

Um token de acesso pode acabar no histórico do navegador se um cliente ou um site que já possui um token navegar deliberadamente para uma página como esta: 

```http
provider.com/get_user_profile?access_token=abcdef 
```

Recomenda a transferência de tokens por meio de um cabeçalho, mas, na prática, os sites costumam passar tokens de acesso em parâmetros de consulta 

No caso de concessão implícita, uma URL como essa :
```http
client.example/redirection_endpoint#access_token=abcdef
```

também pode acabar no histórico do navegador como resultado de um redirecionamento do endpoint de autorização do provedor.

Contramedidas: 

- Os clientes NÃO DEVEM passar tokens de acesso em um parâmetro de consulta URI.
- O método de concessão de código de autorização ou modos de resposta OAuth alternativos, como o modo de resposta POST de formulário, podem ser usados ​​para esse fim.

- [ ] Ameaça: Ataques de Confusão

- [ ] Ameaça: Injeção de código de autorização

- [ ] Ameaça: Injeção de Token de Acesso

- [ ] Falsificação de solicitação entre sites

- [ ] Vazamento de token de acesso no servidor de recursos

- [ ] Ameaça: Vazamento de token de acesso no servidor de recursos

- [ ] Ameaça: Redirecionamento 307

- [ ] Ameaça: Proxies reversos com terminação TLS

- [ ] Ameaça: Cliente se passando pelo proprietário do recurso

- [ ] Ameaça: Clickjacking

#### **Outras Considerações de Segurança**

 - [ ] Confidencialidade dos pedidos

- [ ]  Autenticação do servidor

 - [ ] Mantenha sempre o proprietário do recurso informado.

 - [ ] Credenciais

 - [ ] Proteção de armazenamento de credenciais

 - [ ] Contramedidas padrão para SQLi

 - [ ] Não há armazenamento de credenciais em texto não criptografado.

 - [ ] Criptografia de credenciais

 - [ ] Utilização da criptografia assimétrica

 - [ ] Ataques online a segredos

 - [ ] Política de senhas

 - [ ] Alta entropia dos segredos

 - [ ] Contas de bloqueio

 - [ ] Poço de piche

 - [ ] Utilização de CAPTCHAs

 - [ ] Tokens (acesso, atualização, código)

 - [ ] Limitar o escopo do token

 - [ ] Tempo de validade

 - [ ] Validade curta

 - [ ] Número limitado de utilizações/Utilização única

 - [ ] Vincular tokens a um servidor de recursos específico (Audience)

 - [ ] Usar o endereço do endpoint como público-alvo do token

 - [ ] Escopos de público-alvo e token

 - [ ] Vincular token ao ID do cliente

 - [ ] Tokens assinados

 - [ ] Criptografia do conteúdo do token

 - [ ] Valor aleatório do token com alta entropia

 - [ ] Tokens de acesso

 - [ ] Servidor de autorização

 - [ ] Códigos de autorização

 - [ ] Revogação automática de tokens derivados em caso de detecção de abuso.

 - [ ] Tokens de atualização

 - [ ] Emissão restrita de tokens de atualização

 - [ ] Vinculação do token de atualização ao client_id

 - [ ] Substituição do token de atualização

 - [ ] Revogação do Token de Atualização

 - [ ] Combine solicitações de token de atualização com o segredo fornecido pelo usuário.

 - [ ] Identificação do dispositivo

 - [ ] Autenticação e autorização do cliente

 - [ ] O ID do cliente só pode ser usado em combinação com o consentimento forçado do usuário.

 - [ ] Client_id somente em combinação com redirect_uri

 - [ ] Validação do redirect_uri pré-registrado

 - [ ] Revogação do sigilo do cliente

 - [ ] Utilize autenticação forte do cliente (ex: client_assertion / client_token)

 - [ ] Autorização do usuário final

 - [ ] O processamento automático de autorizações repetidas requer validação do cliente.

 - [ ] Validação das propriedades do cliente pelo usuário final

 - [ ] Vinculação do código de autorização ao client_id

 - [ ] Vinculação do código de autorização ao redirect_uri

#### **Segurança do aplicativo cliente**

- [ ]  Não armazene credenciais em código ou recursos incluídos em pacotes de software.

- [ ]  Armazene segredos em um local seguro.

- [ ] Utilize o bloqueio de dispositivo para impedir o acesso não autorizado ao dispositivo.

#### **Servidores de recursos**

- [ ]  Verificar cabeçalhos de autorização

- [ ] Verificar solicitações autenticadas

- [ ]  Verificar solicitações assinadas







 
