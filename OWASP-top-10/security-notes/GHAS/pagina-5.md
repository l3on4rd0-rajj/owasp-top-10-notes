# Riscos de Ignorar Alertas e Controle de Acesso

## Implicações de ignorar um alerta

Ignorar um alerta de segurança representa riscos significativos para o projeto.

Vulnerabilidades não tratadas podem ser exploradas por agentes maliciosos, levando a:

- violações de dados;
- interrupções de serviço;
- outros incidentes de segurança.

Além disso, ignorar alertas também pode:

- aumentar o esforço de correção no futuro;
- impactar cronogramas do projeto;
- comprometer a confiabilidade geral do software.

## Consequências de longo prazo

O material original destaca que as consequências podem incluir:

- danos à reputação;
- descumprimento de normas regulatórias;
- perdas financeiras.

Por isso, é crucial que as equipes de desenvolvimento priorizem e tratem os alertas de segurança prontamente, a fim de mitigar riscos e manter a integridade do software.

## Papel imediato dos desenvolvedores

Quando um alerta de segurança é identificado, a função imediata da equipe é investigar:

- a natureza do alerta;
- a gravidade;
- o impacto no código-fonte;
- possíveis cenários de exploração;
- as medidas necessárias para correção.

Com a ajuda do GHAS, desenvolvedores podem encontrar e corrigir vulnerabilidades mais cedo no ciclo de vida do desenvolvimento de software.

---

## Quem tem acesso aos alertas?

O gerenciamento de acesso é baseado em funções, com diferentes níveis de permissão para visualizar ou modificar alertas do GHAS.

### Regras destacadas no material

- **Code scanning** e **Dependabot alerts** podem ser visualizados e modificados por pessoas com permissão de escrita ou função administrativa no repositório.
- **Secret scanning alerts** podem ser visualizados e modificados por pessoas com função administrativa no repositório.
- Pessoas ou equipes também podem receber acesso específico para visualizar e modificar alertas por meio das configurações de **Acesso a alertas** do repositório.

## Ponto de atenção

O acesso aos alertas influencia diretamente a velocidade de resposta da equipe. Um modelo de permissões bem configurado evita gargalos e facilita a remediação.
