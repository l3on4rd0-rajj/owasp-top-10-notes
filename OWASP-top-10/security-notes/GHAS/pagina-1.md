# Recursos Essenciais de Segurança no GitHub

Esta página reúne uma visão geral dos principais recursos de segurança apresentados no material original.

## Destaques

- **Secret scanning** ajuda a identificar segredos expostos, como tokens e chaves de API.
- **CodeQL** analisa o código-fonte em busca de vulnerabilidades e erros de codificação.
- **Dependabot** monitora dependências, aponta vulnerabilidades e propõe atualizações.
- A combinação desses recursos torna a segurança parte do fluxo de desenvolvimento, e não uma etapa isolada.

## Secret Scanning

A verificação secreta é um recurso de segurança essencial do GitHub que identifica e ajuda a prevenir a exposição acidental de informações confidenciais, como chaves de API e tokens, no código-fonte.

**A verificação secreta está disponível gratuitamente para todos os repositórios públicos e também pode ser ativada para repositórios privados com uma licença GitHub Advanced Security (GHAS).**

A verificação secreta funciona buscando padrões e assinaturas predefinidos que indiquem informações sensíveis, garantindo que riscos de segurança em potencial sejam prontamente resolvidos.

Por padrão, a verificação secreta busca padrões altamente precisos fornecidos por um Parceiro do GitHub. No entanto, padrões personalizados podem ser criados para outros casos de uso.

### O que a varredura secreta inclui

- **Proteção contra push**: impede vazamentos de segredos de forma proativa, analisando o código no momento do commit e bloqueando o push caso um segredo esteja presente.
- **Alertas e correção no GitHub**: permite visualizar os alertas e corrigi-los sem sair da plataforma.

## CodeQL

O CodeQL analisa o código-fonte em busca de vulnerabilidades de segurança e erros de codificação. Ele emprega técnicas de análise estática para identificar problemas potenciais, como:

- injeção de SQL;
- cross-site scripting;
- estouro de buffer.

Ao fornecer feedback automatizado diretamente no fluxo de trabalho do pull request, a análise de código permite que desenvolvedores corrijam vulnerabilidades no início do processo de desenvolvimento.

## Dependabot

O Dependabot é uma ferramenta automatizada de gerenciamento de dependências, responsável por manter as dependências do projeto atualizadas.

Ele verifica regularmente se há atualizações para bibliotecas e frameworks usados em um projeto e abre automaticamente solicitações de pull request para atualizar as dependências para versões mais recentes e seguras.

### O que o Dependabot inclui

- **Alertas para vulnerabilidades conhecidas**
- **Atualizações de segurança para pacotes vulneráveis**
- **Atualizações de versão para manter dependências atualizadas**

O Dependabot trabalha em estreita colaboração com o **Dependency Graph** para determinar quais dependências estão em uso e compará-las com o **GitHub Advisory Database** para detectar vulnerabilidades.

## Onde ativar esses recursos

Para ativar a verificação de segredos, a verificação de código e os alertas do Dependabot no nível do repositório:

1. Acesse a guia **Security** do repositório.
2. Ative os alertas na visão geral de segurança.
3. Consulte as seções:
   - `dependabot alerts`
   - `code scanning alerts`
   - `secret scanning alerts`

## Impacto combinado dessas funcionalidades

> Ao integrar esses recursos, as equipes de desenvolvimento podem abordar proativamente as preocupações com segurança em todas as etapas do ciclo de vida do desenvolvimento.

Essa abordagem proativa:

- minimiza a probabilidade de incidentes de segurança chegarem à produção;
- torna o processo de desenvolvimento mais resiliente;
- fortalece a eficiência operacional;
- incorpora segurança ao fluxo normal de trabalho.

## Projetos de código aberto x GHAS em repositórios privados

Projetos públicos no GitHub já se beneficiam de alguns recursos de segurança por padrão, como:

- verificação de segredos;
- gráfico de dependências.

Além disso, projetos públicos também podem optar por habilitar:

- verificação de código;
- Dependabot;
- revisão de dependências.

Sem GHAS, esses recursos podem não oferecer a profundidade de proteção necessária para projetos mais complexos ou ambientes corporativos.

Quando o **GitHub Advanced Security (GHAS)** é combinado com o **GitHub Enterprise Cloud (GHEC)**, o conjunto de segurança fica disponível também para projetos internos e privados.

### Recursos destacados nesse cenário

- **Análise de código**: busca por possíveis vulnerabilidades de segurança e erros de codificação.
- **Varredura de segredos**: detecta segredos, como chaves e tokens, armazenados em repositórios privados.
- **Análise de dependências**: mostra o impacto das alterações em dependências e ajuda a identificar versões vulneráveis antes do merge.

## Distribuição do conteúdo nas próximas páginas

Os tópicos a seguir foram separados para melhorar a leitura do material:

- [pagina-2.md](./pagina-2.md): gerenciamento de dependências e gráfico de dependências;
- [pagina-3.md](./pagina-3.md): como utilizar o GHAS para obter mais impacto;
- [pagina-4.md](./pagina-4.md): tipos de alertas do GHAS;
- [pagina-5.md](./pagina-5.md): riscos de ignorar alertas e controle de acesso.
