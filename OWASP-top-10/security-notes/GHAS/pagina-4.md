# Agindo com Base nos Alertas do GHAS

Esta página organiza os tipos de alertas mencionados no material original e mostra por que eles são importantes no fluxo de segurança.

## Tipos de alertas

### Alertas de leitura de código

#### Alertas de análise do CodeQL

Gerados pelo CodeQL, mecanismo de análise semântica de código do GitHub, esses alertas identificam possíveis vulnerabilidades de segurança na base de código.

Eles podem abranger problemas como:

- injeção de SQL;
- cross-site scripting;
- outras vulnerabilidades de código.

### Alertas de varredura secreta

#### Alertas de segredos expostos

Esses alertas são acionados quando informações potencialmente sensíveis, como chaves de API ou credenciais, são identificadas no código-fonte do repositório.

A verificação de segredos ajuda a evitar a exposição acidental de dados confidenciais.

### Alertas de dependência

#### Alertas do Dependabot

O Dependabot detecta automaticamente dependências desatualizadas em um projeto e pode criar solicitações de pull request para atualizá-las para versões mais recentes e seguras.

Os alertas do Dependabot notificam os desenvolvedores sobre atualizações disponíveis para as dependências do projeto.

### Alertas da Visão Geral de Segurança

A Visão Geral de Segurança fornece um painel abrangente que resume o status de segurança do repositório.

### Alertas de terceiros

Ferramentas de análise de código de terceiros podem ser integradas à verificação de código do GitHub por meio do envio de dados em arquivos **SARIF**.

## Leitura rápida

Se a equipe quiser responder mais cedo aos riscos, os alertas do GHAS ajudam a enxergar rapidamente:

- vulnerabilidades no código;
- segredos expostos;
- dependências vulneráveis ou desatualizadas;
- a postura geral de segurança do repositório.
