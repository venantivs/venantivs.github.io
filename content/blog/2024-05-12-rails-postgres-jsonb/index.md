+++
title = "Utilizando JSONB com Rails"
date = 2024-05-12
draft = false

[taxonomies]
categories = ["rails"]
tags = ["ruby", "rails", "postgres", "jsonb"]

[extra]
lang = "pt-BR"
toc = true
comment = true
copy = true
math = false
mermaid = false
outdate_alert = false
outdate_alert_days = 120
display_tags = true
truncate_summary = true
featured = false
+++

Veja como utilizar, indexar, e evitar queries N+1 com os campos do tipo JSONB no Ruby on Rails.

<!-- more -->

## Introdução

Às vezes, precisamos guardar o retorno de uma API, _logs_, ou simplesmente queremos guardar dados no Postgres de uma forma mais maleável, sem precisar apelar para alterações no _schema_ da tabela quando um atributo novo precisar ser inserido ou removido. Para isso, o Postgres nos fornece os tipos `JSON` e `JSONB`, que iremos falar especialmente sobre o segundo no decorrer deste _post_.

{% note(header="Nota") %}
Enquanto eu escrevia este _post_, acabei me deparando com outro, também em português sobre exatamente o mesmo assunto: [Blog do Nando Vieira](https://nandovieira.com.br/usando-postgresql-e-jsonb-no-rails).
{% end %}

## JSON vs JSONB

Se você deseja guardar objetos _json_ no Postgres, basicamente você tem duas opções: JSON e JSONB, introduzidos nas versões 9.2 e 9.4 do Postgres, respectivamente. A diferença mais relevante entre ambas é que o tipo JSONB pode ser indexado, o que pra quem lida com uma quantidade razoável de dados pode fazer MUITA diferença.

Outras diferenças menores incluem que o tipo JSON, na inserção, insere os dados exatamente como foram fornecidos, não fazendo qualquer tipo de conversão. A consequência disto é que a ordem das chaves dentro dos objetos, chaves repetidas, e espaços em branco sintaticamente irrelevantes (espaço entre a chave e o `:`, por exemplo) serão mantidos. Porém, apesar disto significar um tempo de inserção menor, a conversão que não foi preciso ser feita na inserção precisará ser feita em qualquer outra operação necessária sobre o campo (incluindo projeção), o que também significará um tempo maior de processamento. O JSONB, por outro lado, logo na inserção é feita sua conversão para uma estrutura binária, que não precisará de conversão em outras operações, tornando-a mais performática para esses casos.

Um outro ponto que deve ser levado em consideração é que o campo JSONB possui mais operadores que o campo JSON.

Mais informações [aqui](https://www.postgresql.org/docs/current/datatype-json.html).

### Em que situação usar JSON invés de JSONB?

A menos que a ordem das chaves dentro dos objetos importe para seu uso, ou que você raramente irá utilizar qualquer operação sobre o campo que não seja inserção, como _logs_ por exemplo, minha opinião é que não há motivos para usar o tipo JSON. Ainda assim, a não ser que seu requisito seja manter a ordem das chaves, JSONB pode funcionar muito bem nos outros cenários também.

## Operadores

Antes de irmos para o código, é importante aprender e entender a sintaxe que torna as operações com _json_ possíveis no Postgres.

### Operadores JSON e JSONB

Os operadores a seguir estão disponíveis tanto para JSON quanto para JSONB e provavelmente são os mais comuns e mais utilizados, pois são utilizados para fazer pesquisas.

O importante aqui é entender que a estrutura é sempre um JSON/JSONB, provavelmente a coluna da tabela, com `->`, e depois vem um texto ou número inteiro. Caso seja um texto, é extraído o elemento cuja chave é o texto. Caso seja um inteiro, é extraído o elemento que se encontra no índice representado pelo inteiro. No caso do operador `#>`, à direita do operador colocamos um texto com um caminho a ser acessado no _json_ (e.g. `{a,b}`). Ainda, pode ser adicionado mais um `>` ao operador (e.g. `->>` ou `#>>`) fazendo com que o retorno seja texto, ao invés do tipo do operador à esquerda (JSON ou JSONB).

| Operador                        | Descrição                                                                                              |
|---------------------------------|--------------------------------------------------------------------------------------------------------|
| jsonb ->[>] text \| integer     | Extrai o elemento na posição "integer" ou com chave "text".                                            |
| jsonb #>[>] text[]              | Extrai o elemento no caminho especificado.                                                             |

Outra característica importante é que utilizando o primeiro operador com retorno JSON/JSONB (`->`), ele é encadeável. Isto é, em caso de um objeto _json_ de vários níveis, você poderia fazer, por exemplo:

```
'{"a": {"b": "olá" }}'::jsonb -> a ->> b → "olá"
```

Ou, obtendo o mesmo resultado utilizando `#>>` especificando o caminho:

```
'{"a": {"b": "olá" }}'::jsonb #>> '{a,b}' → "olá"
```

### Operadores apenas JSONB

Apenas para JSONB, há uma lista maior de operadores, que não irei cobrir inteira aqui (apesar de aparecem nos exemplos) para que este _post_ não fique maior do que já está. A lista completa pode ser acessada [aqui](https://www.postgresql.org/docs/16/functions-json.html#FUNCTIONS-JSONB-OP-TABLE). Entretanto, falaremos de dois operadores específicos do JSONB que são importantes para pesquisas em objetos JSON: `@>` (ou `<@`) e `?`.

| Operador       | Descrição                                               |
|----------------|---------------------------------------------------------|
| jsonb @> jsonb | Checa se json à esquerda contém o da direita.           |
| jsonb ? text   | Checa se o elemento "text" existe (no nível mais raso). |

## JSONB no Rails

{% warning(header="Atenção") %}
O suporte para JSONB no Rails foi adicionado na versão 4.2
{% end %}

### Adicionando um campo JSONB

Para adicionar um campo JSONB ao criar uma tabela no Rails, basta colocar na _migration_:

```ruby
create_table :events do |t|
  # ...
  t.jsonb 'payload'
  # ...
end
```

Ou, para adicionar em uma tabela já existente:

```ruby
def change
  add_column :events, :payload, :jsonb, default: {}
end
```

O parâmetro `default` é opcional, assim como em outros tipos.

### Criando registros com JSON

Para criar um registro em um modelo que possua um atributo com JSONB (ou JSON), basta passar o campo como Hash, que o próprio ActiveRecord irá fazer a conversão:

```ruby
Event.create!(
  # ...
  payload: {
    a: {
      b: {
        c: "fizz"
      }
    },
    foo: "bar",
    array: [1,2,3,4]
  }
  # ...
)
```

Uma vez criado, para acessar o campo, o ActiveRecord também converte o objeto _json_ em _hash_:

```ruby
event = Event.last.payload
# => {"a"=>{"b"=>{"c"=>"fizz"}}, "foo"=>"bar", "array"=>[1,2,3,4]}
```

Algo a ser notado aqui, é que apesar de termos passado uma _hash_ com as chaves como símbolos, o retorno sempre será feito com as chaves em _string_, assim como manda a [especificação do JSON](https://datatracker.ietf.org/doc/html/rfc7159#section-4):

```ruby
event = Event.last

event.payload['foo']
# => "bar"

event.payload[:foo]
# => nil
```

Caso prefira, como o campo no Rails se tornará uma _hash_, nada te impede de utilizar métodos como `deep_symbolize_keys` ou `with_indifferent_access` para acessar os objetos utilizando símbolos. Apenas tenha em mente que terá de fazer isto toda vez que um registro for carregado do banco.

Outro ponto de atenção é que você deve ter cuidado com quais tipos de dados você passa na Hash que será convertida em _json_. Cada chave deve conter ou outra _hash_, ou um _array_, um número ou uma _string_. Qualquer coisa fora disso será convertida em _string_ e quando voltar ao Ruby, será mantida como _string_. Em casos de objetos Ruby, estes serão serializados. Exemplo:

```ruby
class Thing
  def initialize(a, b)
    @a = a
    @b = b
  end
end

Event.create!({
  payload: {
    current_time: Time.current
    thing: Thing.new(1, 2)
  }
})

Event.last.payload['current_time'].class
# => String

Event.last.payload['thing']
# => {"a" => 1, "b" => 2}
```

### Alterando registros

Para alterar registros com campo JSONB, basta alterar a Hash e mandar salvar o registro:

```ruby
event = Event.last

event.payload['foo'] = 'baz'

event.save!
```

ou, utilizando o método `update`:

```ruby
event = Event.last

new_payload = event.payload.merge({ 'foo' => 'baz' })

event.update!(payload: new_payload)
```

A partir disso, como estamos alterando o campo inteiro, a partir da manipulação da `Hash` conseguimos fazer qualquer coisa: Adicionar ou remover chaves, alterar valores, etc.

### Alterando registros em massa

Se você é atento, reparou que na subseção anterior, de ambas as formas, acabamos por carregar o registro inteiro para a aplicação e substituímos todo o campo JSONB, mesmo que alteramos apenas um atributo. Apesar de funcionar, pode não ser o ideal em casos onde objeto _json_ é muito grande ou seja necessário alterar vários registros de uma vez (criando uma situação de Query N+1).

#### Alterando um atributo

Para alterar apenas um atributo, precisaremos utilizar uma função específica do Postgres, a `jsonb_set`.

```ruby
Event.where(...).update_all("payload = jsonb_set(payload, '{foo}', to_jsonb('baz'::text))")
```

#### Inserindo um novo atributo

Na mesma linha, ao invés de atualizar um valor de um atributo, você pode também querer apenas adicionar um novo atributo junto com um valor. Para isso existe a função `jsonb_insert`:

```ruby
Event.where(...).update_all("payload = jsonb_insert(payload, '{hello}', to_jsonb('world'::text))")
```

A função `jsonb_insert` também pode ser utilizada para inserir um elemento em um _array_:

```ruby
Event.where(...).update_all("payload = jsonb_insert(payload, '{array, 0}', to_jsonb('new_value'::text))")
```

No caso acima, estamos inserindo "new_value" na primeira posição do _array_.

#### Removendo um atributo 

Para remover um atributo junto com seu valor, podemos utilizar o operador `-` (cuidado ao utilizá-lo):

```ruby
Event.where(...).update_all("payload = payload - 'foo'")
```

### Realizando queries

Até agora criamos e atualizamos registros. E para encontrar um registro específico dependendo de um ou mais atributos _json_? Aqui vão alguns casos comuns:

#### Retornar registros em que um atributo é igual a um valor

```ruby
Event.where("payload->>'foo' = ?", 'bar')
```

Ou utilizando o operador com retorno em jsonb:

```ruby
Event.where("payload->'foo' = ?", 'bar'.to_json)
``` 

Podemos também utilizar o operador "contém":

```ruby
User.where("payload @> ?", { foo: 'bar' }.to_json)
```

#### Retornar registros cujo atributo aninhado é igual um valor

```ruby
Event.where("payload->'a'->'b'->>'c' = ?", 'fizz')
```

Ou utilizando o operador com caminho:

```ruby
Event.where("payload #>> '{a,b,c}' = ?", "fizz")
```

#### Retornar registros que possuem uma chave específica definida

```ruby
Event.where("payload ? :key", key: 'foo')
```

Caso seja necessário checar se mais de uma chave existem ao mesmo tempo, podemos utilizar o operador `?&`

```ruby
Event.where("payload ?& array[:keys]", keys: ['foo', 'a'])
```

Também há a possibilidade de, dado várias opções, retornar os registros que possuem pelo menos uma das chaves utilizando o operador `?|`

```ruby
Event.where("payload ?& array[:keys]", keys: ['foo', 'any_other_key'])
```

#### Retornar registros que contém um determinado objeto

```ruby
Event.where("extradata @> ?", {a: {b: {c: "fizz"}}}.to_json)
```

## Performance

{% note(header="Nota") %}
Os testes foram feitos utilizando Ruby 3.3.0, Rails 7.1.3.2 e Postgres 16.2
{% end %}

A fim de tentar demonstrar melhor as diferenças entre JSON e JSONB, criei duas tabelas diferentes, ambas com 100.000 (cem mil) registros, cada uma com um objeto json simples (nome gerado a cada inserção pela _gem_ [Faker](https://github.com/faker-ruby/faker)):

```json
{
  "user": {
    "name": "RANDOMLY GENERATED"
  }
}
```

```ruby
EventsJson.count
=> 100000
EventsJsonb.count
=> 100000
```

Comparando ambas, sem um índice no campo JSONB, pesquisando 1.000 vezes uma string estática inexistente:

```
Rehearsal -----------------------------------------
json    0.249155   0.114169   0.363324 ( 66.338925)
jsonb   0.282361   0.028095   0.310456 ( 17.515571)
-------------------------------- total: 0.673780sec

            user     system      total        real
json    0.295434   0.061096   0.356530 ( 65.830411)
jsonb   0.243282   0.072838   0.316120 ( 17.484288)
```

Nota-se que o jsonb é consideravelmente mais rápido, em torno de 73%, o que pode ser atribuído ao fato do Postgres não precisar fazer a conversão do campo a cada operação. Porém, conforme o número de registros aumentar nestas duas tabelas, a chance é que essa diferença diminua com o tempo.

"Rodando" ambas as queries com EXPLAIN, é confirmado que ambas estão fazendo uma busca sequencial (Seq Scan):

JSON:
```
# EXPLAIN SELECT * FROM events_jsons WHERE payload->'user'->>'name' = 'abcedefgh'

Seq Scan on events_jsons  (cost=0.00..2877.67 rows=500 width=59)
  Filter: (((payload -> 'user'::text) ->> 'name'::text) = 'abcdefgh'::text)
```
JSONB:
```
# EXPLAIN SELECT * FROM events_jsonbs WHERE payload->'user'->>'name' = 'abcedefgh'

Seq Scan on events_jsonbs  (cost=0.00..3021.28 rows=500 width=71)
  Filter: (((payload -> 'user'::text) ->> 'name'::text) = 'abcdefgh'::text)
```

### Indexando campos JSONB

Para indexar campos JSONB, você basicamente tem 3 opções de tipos de índice: `b-tree`, `hash` ou `gin`.

#### Qual tipo de índice utilizar?

Caso você queira indexar o campo inteiro, sua melhor opção será o índice do tipo `gin`. Os tipos `b-tree` e `hash`, caso utilizados no campo inteiro, só são úteis para comparar o objeto json inteiro.

Ainda assim, o tipo `gin` possui suas limitações. A principal que deve ser levada em consideração é que caso for indexado o campo todo, os operadores padrão apenas irão utilizar o índice caso a condição da _query_ esteja pesqusiando por atributos no nível "mais raso" do objeto _json_. Para driblar este problema, um índice de expressão deve ser criado ou deve ser feita a utilização do operador `@>`, com o operando à direita devendo ser um objeto _json_. Outra limitação importante é que índices do tipo `gin` não suportam operadores como `=`, `>`, `>=`, `<`, `<=`, etc. Caso você precise, por exemplo, checar se o valor em uma chave é maior que determinado outro, uma saída melhor é utilizar um índice de expressão do tipo `b-tree` no "caminho" desejado do objeto _json_.

Este pequeno trecho acima é a "ponta do _iceberg_" sobre indexação de campos JSONB. O importante aqui é não sair criando índices cegamente sem antes analisar qual será o perfil das _queries_ feitas. Para mais informações e exemplos (em inglês): [Documentação do Postgres](https://www.postgresql.org/docs/current/datatype-json.html#JSON-INDEXING) ou [este ótimo artigo](https://scalegrid.io/blog/using-jsonb-in-postgresql-how-to-effectively-store-index-json-data-in-postgresql/).

### Criando um índice GIN no Rails

Como o objeto json que criamos é aninhado, para que o índice seja utilizado ao buscarmos pelo nome do usuário, podemos tanto criar um índice `gin` no campo inteiro e utilizar o operador `@>`, ou criar um índice de expressão (este podendo ser `b-tree` ou `gin`). Para começar, vamos indexar o campo inteiro utilizando o tipo `gin`:

```
class AddIndexToEvents < ActiveRecord::Migration[7.1]
  def change
    add_index :events_jsonbs, :payload, using: :gin, name: 'index_events_jsonbs_on_payload'
  end
end
```

Agora, vamos rodar um `EXPLAIN` na nossa query novamente para ver se o índice está sendo utilizado:

```
# EXPLAIN SELECT * FROM events_jsonbs WHERE payload->'user'->>'name' = 'abcedefgh'

Seq Scan on events_jsonbs  (cost=0.00..3021.00 rows=500 width=71)
  Filter: (((payload -> 'user'::text) ->> 'name'::text) = 'abcedefgh'::text)
```

E... Continuamos utilizando a busca sequencial. Como dito anteriormente, índices do tipo GIN não suportam operadores como `=`, então, precisamos trocar por `@>`:

```
# EXPLAIN SELECT * FROM events_jsonbs WHERE payload @> '{"user": {"name": "abcdefgh"}}'

Bitmap Heap Scan on events_jsonbs  (cost=43.06..80.53 rows=10 width=71)
  Recheck Cond: (payload @> '{"user": {"name": "abcdefgh"}}'::jsonb)
  ->  Bitmap Index Scan on index_events_jsonbs_on_payload  (cost=0.00..43.06 rows=10 width=0)
        Index Cond: (payload @> '{"user": {"name": "abcdefgh"}}'::jsonb)
```

Também foi dito que índices do tipo `gin` apenas indexam o nível mais raso (quando não utilizado um índice de expressão), então ainda usando o operador `@>`, porém acessando um objeto mais aninhado, o índice também não funciona:

```
# EXPLAIN SELECT * FROM events_jsonbs WHERE payload->'user' @> '{"name": "abcdefgh"}'

Seq Scan on events_jsonbs  (cost=0.00..2771.00 rows=1000 width=71)
  Filter: ((payload -> 'user'::text) @> '{"name": "abcdefgh"}'::jsonb)
```

### Criando um índice de expressão B-TREE

Ainda considerando o caso onde queiramos buscar um usuário com nome "abcdefgh", como já dito anteriormente, em índices do tipo `b-tree` só será possível verificar isso com índices de expressão.

{% warning(header="Atenção") %}
O suporte para índices de expressão foi adicionado na versão 5 do Rails, caso você esteja em uma versão anterior, terá de utilizar o método `execute` do `ActiveRecord::Migration`, invés do `add_index` padrão.
{% end %}

Como índices do tipo `b-tree` são padrão, podemos omitir o atributo `using`:

```ruby
class AddBTreeIndexToEvents < ActiveRecord::Migration[7.1]
  def change
    add_index :events_jsonbs, "(payload->'user'->>'name')", name: 'index_events_jsonbs_on_payload_user_name'
  end
end
```

Rodando a _query_ novamente:

```
# EXPLAIN SELECT * FROM events_jsonbs WHERE payload->'user'->>'name' = 'abcedefgh'

Bitmap Heap Scan on events_jsonbs  (cost=16.29..977.90 rows=500 width=71)
  Recheck Cond: (((payload -> 'user'::text) ->> 'name'::text) = 'abcedefgh'::text)
  ->  Bitmap Index Scan on index_events_jsonbs_on_payload_user_name  (cost=0.00..16.17 rows=500 width=0)
        Index Cond: (((payload -> 'user'::text) ->> 'name'::text) = 'abcedefgh'::text)
```

Porém, se você prestar atenção, criamos o índice com o último atributo do caminho como texto `->>`, isto quer dizer que o índice foi criado levando isso em consideração, se tentarmos buscar com o operador que retorna JSON/JSONB (`->`), o índice não será utilizado:

```
# EXPLAIN SELECT * FROM events_jsonbs WHERE payload->'user'->'name' = to_jsonb('abcedefgh'::text)

Seq Scan on events_jsonbs  (cost=0.00..3271.00 rows=500 width=71)
  Filter: (((payload -> 'user'::text) -> 'name'::text) = to_jsonb('abcedefgh'::text))
```

### Comparando resultados

Ao rodar o _benchmark_ novamente com 1.000 iterações, obtivemos o seguinte resultado:

```
Rehearsal -----------------------------------------------------------------
json                            0.227027   0.121803   0.348830 ( 65.469822)
jsonb b-tree expression index   0.071625   0.018892   0.090517 (  0.198652)
jsonb gin index                 0.062371   0.025346   0.087717 (  0.226455)
-------------------------------------------------------- total: 0.527064sec

                                    user     system      total        real
json                            0.286155   0.062060   0.348215 ( 65.596319)
jsonb b-tree expression index   0.053768   0.026578   0.080346 (  0.182883)
jsonb gin index                 0.041516   0.043382   0.084898 (  0.222005)
```

Notamos que em ambos os casos onde o índice foi aplicado houve redução de quase 99% do tempo se comparado ao campo JSONB sem índice (_benchmark_ anterior).

## Conclusão

O uso do campo JSONB no Rails, apesar do suporte, só revela seu total poder se utilizado em conjunto com as funções e operadores nativos do Postgres, com um ponto de atenção especial ao comportamento dos diferentes tipos de índices em um campo JSONB.