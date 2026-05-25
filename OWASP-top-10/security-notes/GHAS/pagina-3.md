# Como Utilizar o GHAS para Obter o Máximo Impacto

## Objetivos desta parte

Nesta seção, o material original destaca três frentes principais:

- entender o gráfico de dependências;
- agir com base nos alertas do GHAS;
- compreender quem tem acesso aos alertas.

## Entendendo o gráfico de dependências

O gráfico de dependências é fundamental para a segurança da cadeia de suprimentos. Ele identifica:

- todas as dependências **upstream**;
- as dependências **downstream públicas** de um repositório ou pacote.

Você pode visualizar as dependências do repositório e algumas de suas propriedades, como informações sobre vulnerabilidades, no gráfico de dependências do próprio repositório.

### Como o GitHub gera esse gráfico

Para gerar o gráfico de dependências, o GitHub analisa as dependências explícitas de um repositório, declaradas:

- no manifesto;
- nos arquivos de bloqueio.

Quando ativado, o gráfico analisa automaticamente todos os arquivos de manifesto de pacotes conhecidos no repositório e usa essa análise para construir um gráfico com:

- nomes das dependências;
- versões conhecidas.

### Pontos principais sobre o gráfico de dependências

- Inclui informações sobre dependências **diretas** e **transitivas**.
- É atualizado automaticamente quando um commit altera ou adiciona um arquivo de manifesto ou de bloqueio compatível na branch padrão.
- Também pode ser atualizado quando alguém envia uma alteração para o repositório de uma dependência usada pelo projeto.
- Pode ser acessado na página principal do repositório, pela aba **Insights**.
- Pode ser exportado como uma **SBOM compatível com SPDX**, desde que o usuário tenha pelo menos acesso de leitura ao repositório.

## Como esse gráfico é aproveitado pelo GitHub

O material relaciona o gráfico de dependências a três recursos importantes:

### Análise de dependências

Utiliza o gráfico de dependências para identificar alterações nas dependências e ajudar a entender o impacto dessas mudanças na segurança ao revisar pull requests.

### Alertas do Dependabot

O Dependabot cruza os dados fornecidos pelo gráfico de dependências com os avisos publicados no **GitHub Advisory Database**. Quando uma vulnerabilidade potencial é detectada, o GitHub gera alertas.

### Atualizações de segurança do Dependabot

Usam o gráfico de dependências e os alertas do Dependabot para ajudar a atualizar dependências com vulnerabilidades conhecidas.

## Observação importante

Embora as **atualizações de versão do Dependabot** não usem diretamente o gráfico de dependências, elas continuam relevantes: ajudam a manter as dependências atualizadas com base em versionamento semântico, mesmo quando não há vulnerabilidades conhecidas.
