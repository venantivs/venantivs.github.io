+++
title = "Rails: Queries N+1, suas consequências e como resolvê-las"
date = 2024-08-08
draft = false

[taxonomies]
categories = ["rails"]
tags = ["ruby", "rails", "active record", "performance", "n+1 queries"]

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

Queries N+1 são provavelmente o maior responsável por perda de desempenho em aplicações Rails, veja neste post como evitar e resolvê-las.

<!-- more -->

## Introdução

Quando estamos desenvolvendo aplicações Rails, dependendo da quantidade de acessos e usuários que a aplicação irá receber, é importante alocar recursos de desenvolvimento focando no desempenho da aplicação. Um dos principais problemas que assolam aplicações Rails, quando o assunto é desempenho, são as _queries_ N+1. Neste _post_ iremos aprofundar sobre o assunto, cobrindo métodos de detecção, _debugging_ e solução de diversas possibilidades.

## O que são Queries N+1?

Queries N+1, em suma, ocorrem quando a aplicação realiza uma _query_ para trazer um registro, e depois realiza N _queries_, uma por cada registro associado.

Para entender melhor, vamos supor um caso comum: Posts e Comentários. Um Blog possui N Posts, e cada Post possui N comentários associados.

{{ figure(src="posts_comments.jpg", alt="alt text", caption="DER de um Blog simples") }}

```ruby
# models/post.rb

class Post < ApplicationRecord
  has_many :comments
end
```

```ruby
# models/comment.rb

class Comment < ApplicationRecord
  belongs_to :post
end
```

Agora, supondo que queiramos recuperar os últimos 5 _posts_ feitos e seus respectivos comentários, imprimindo na tela cada título post e o primeiro comentário, poderíamos fazer:

```ruby
Post.last(5).each { |post| puts post.comments.first&.body }
```

Porém, ao rodar, vemos que foi feita uma _query_ para buscar os comentários de cada post:

```ruby
irb(main):030> Post.last(5).each { |post| puts post.comments.first&.body };0
  Post Load (0.5ms)  SELECT "posts".* FROM "posts" ORDER BY "posts"."id" DESC LIMIT $1  [["LIMIT", 5]]
  Comment Load (0.5ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = $1 ORDER BY "comments"."id" ASC LIMIT $2  [["post_id", 1], ["LIMIT", 1]]
Comentário 1 do Post 1
  Comment Load (0.4ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = $1 ORDER BY "comments"."id" ASC LIMIT $2  [["post_id", 2], ["LIMIT", 1]]
Comentário 1 do Post 2
  Comment Load (0.3ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = $1 ORDER BY "comments"."id" ASC LIMIT $2  [["post_id", 3], ["LIMIT", 1]]
Comentário 1 do Post 3
  Comment Load (0.4ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = $1 ORDER BY "comments"."id" ASC LIMIT $2  [["post_id", 4], ["LIMIT", 1]]
Comentário 1 do Post 4
  Comment Load (0.4ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = $1 ORDER BY "comments"."id" ASC LIMIT $2  [["post_id", 5], ["LIMIT", 1]]
Comentário 1 do Post 5
=> 0
```

### Por que isso ocorre?

O problema de _query_ N+1 acontece porque o ActiveRecord utiliza uma técnica chamada _lazy loading_. Esta técnica consiste em NÃO carregar uma entidade do banco de dados até o momento em que essa entidade é acessada.

Se, por exemplo, fizermos:

```ruby
post = Posts.find(1)
```

Enquanto a variável `post` não for lida, a requisição para o banco de dados não será feita. O mesmo acontece com as associações:

```ruby
post.comments
```

Enquanto não tentarmos acessá-las, não serão feitas requisições no banco.

É importante notar também que há algumas exceções, como por exemplo, quando utilizamos `to_a` em uma query.

{% note(header="Nota") %}
O console do Rails automaticamente faz um `inspect` ao fim de cada *statement*, por isso que você verá ele fazendo as requisições antes de você tentar ler as variáveis associadas. Para ver o _lazy loading_ em funcionamento no console, coloque um `; nil` ao final de cada *statement*.
{% end %}

Para entender melhor o funcionamento do _lazy loading_, leia [este artigo](https://www.rubyinrails.com/2014/01/08/what-is-lazy-loading-in-rails/).

### Por que isso é um problema?

Toda vez que o ActiveRecord executa uma *query*, uma conexão com o banco de dados é aberta, a *query* é executada, e depois a conexão é fechada. O problema com isso é que bancos de dados geralmente não residem "fisicamente" na mesma máquina que a aplicação, logo, a cada *query* uma latência extra para conexão/desconexão é adicionada. Apesar desta latência não ser exatamente grande, ela não é desprezível, principalmente se acumulado pela execução de várias *queries* consecutivas. Não é incomum ver casos de *query* N+1 onde a piora no desempenho chega a ser geométrica.

{% note(header="Nota") %}
Apesar de estarmos falando especificamente do *framework* Ruby on Rails neste _post_, é importante frisar que este é um problema presente em praticamente qualquer ORM, não apenas o ActiveRecord.
{% end %}

## Resolvendo Queries N+1

Ainda considerando o caso anterior, aparece uma dúvida: Quantas queries devem ser feitas para que o mesmo resultado seja atingido? E aqui temos duas possibilidades:

1. Apenas uma _query_ com LEFT JOIN;
2. Duas _queries_, uma para o Post e outra para os comentários.

### Método eager_load

Relativo às duas possibilidades enumeradas anteriormente, o método `eager_load` se refere à primeira. Toda vez que utilizado, será feito um **LEFT OUTER JOIN**. Utilizando o mesmo exemplo anterior, vemos que apenas uma _query_ é feita:

```ruby
irb(main):003> Post.eager_load(:comments).all.each { |post| puts post.comments.first&.body };0
  SQL (1.2ms)  SELECT "posts"."id" AS t0_r0, "posts"."title" AS t0_r1, "posts"."body" AS t0_r2, "posts"."created_at" AS t0_r3, "posts"."updated_at" AS t0_r4, "comments"."id" AS t1_r0, "comments"."body" AS t1_r1, "comments"."post_id" AS t1_r2, "comments"."created_at" AS t1_r3, "comments"."updated_at" AS t1_r4 FROM "posts" LEFT OUTER JOIN "comments" ON "comments"."post_id" = "posts"."id"
Comentário 1 do Post 1
Comentário 1 do Post 2
Comentário 1 do Post 3
Comentário 1 do Post 4
Comentário 1 do Post 5
```

### Método preload
O método `preload` se refere à segunda opção. Para cada associação passada para o método, uma _query_ adicional será feita, além da _query_ original:

```ruby
irb(main):004> Post.preload(:comments).all.each { |post| puts post.comments.first&.body };0
  Post Load (2.3ms)  SELECT "posts".* FROM "posts"
  Comment Load (1.0ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" IN ($1, $2, $3, $4, $5)  [["post_id", 1], ["post_id", 2], ["post_id", 3], ["post_id", 4], ["post_id", 5]]
Comentário 1 do Post 1
Comentário 1 do Post 2
Comentário 1 do Post 3
Comentário 1 do Post 4
Comentário 1 do Post 5
```

### Método includes

E o método `includes`? Este método é um método "extra" que decide por si só qual dos dois métodos anteriores será utilizado:

```ruby
irb(main):005> Post.includes(:comments).all.each { |post| puts post.comments.first&.body };0
  Post Load (2.5ms)  SELECT "posts".* FROM "posts"
  Comment Load (0.9ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" IN ($1, $2, $3, $4, $5)  [["post_id", 1], ["post_id", 2], ["post_id", 3], ["post_id", 4], ["post_id", 5]]
Comentário 1 do Post 1
Comentário 1 do Post 2
Comentário 1 do Post 3
Comentário 1 do Post 4
Comentário 1 do Post 5
```

No exemplo acima, vimos que, para este caso, o método `includes` escolheu a aborgadem com `preload` ao invés de `eager_load`.

#### Como o método includes decide?

O método `includes` usará o método `preload` sempre que possível, isto é, quando não houver uma cláusula `where` referenciando a associação que está sendo pré-carregada:

```ruby
irb(main):006> Post.includes(:comments).where(comments: { post_id: 1 }).each { |post| puts post.comments.first&.body };0
  SQL (34.9ms)  SELECT "posts"."id" AS t0_r0, "posts"."title" AS t0_r1, "posts"."body" AS t0_r2, "posts"."created_at" AS t0_r3, "posts"."updated_at" AS t0_r4, "comments"."id" AS t1_r0, "comments"."body" AS t1_r1, "comments"."post_id" AS t1_r2, "comments"."created_at" AS t1_r3, "comments"."updated_at" AS t1_r4 FROM "posts" LEFT OUTER JOIN "comments" ON "comments"."post_id" = "posts"."id" WHERE "comments"."post_id" = $1  [["post_id", 1]]
Comentário 1 do Post 1
```

Caso não haja referência às associações pré-carregadas na cláusula `where`, o método `preload` será utilizado:

```ruby
irb(main):008> Post.includes(:comments).where(title: 'Post 1').each { |post| puts post.comments.first&.body };0
  Post Load (1.1ms)  SELECT "posts".* FROM "posts" WHERE "posts"."title" = $1  [["title", "Post 1"]]
  Comment Load (1.1ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = $1  [["post_id", 1]]
Comentário 1 do Post 1
```

Dito isso, já podemos inferir que o método `preload` não funciona com uma clásula `where` filtrando a associação que está sendo pré-carregada:

```ruby
# Inválido
irb(main):011> Post.preload(:comments).where(comments: { post_id: 1 }).each { |post| puts post.comments.first&.body };0
(irb):11:in `<main>': PG::UndefinedTable: ERROR:  missing FROM-clause entry for table "comments" (ActiveRecord::StatementInvalid)
LINE 1: SELECT "posts".* FROM "posts" WHERE "comments"."post_id" = $...
```

Outro detalhe importante é que, a cláusula `where`, neste caso, só funcionará caso o que for passado seja uma *hash*. Caso deseje utilizar fragmentos de SQL (passar uma *string* com SQL puro) na cláusula `where`, será necessário utilizar o método `references` para "forçar" que as associações sejam feitas:

```ruby
# Inválido
irb(main):009> Post.includes(:comments).where('comments.post_id = 1 ').each { |post| puts post.comments.first&.body };0
  Post Load (12.9ms)  SELECT "posts".* FROM "posts" WHERE (comments.post_id = 1 )
(irb):9:in `<main>': PG::UndefinedTable: ERROR:  missing FROM-clause entry for table "comments" (ActiveRecord::StatementInvalid)
LINE 1: SELECT "posts".* FROM "posts" WHERE (comments.post_id = 1 )`

# Válido
irb(main):010> Post.includes(:comments).where('comments.post_id = 1 ').references(:comments).each { |post| puts post.comments.first&.body };0
  SQL (8.9ms)  SELECT "posts"."id" AS t0_r0, "posts"."title" AS t0_r1, "posts"."body" AS t0_r2, "posts"."created_at" AS t0_r3, "posts"."updated_at" AS t0_r4, "comments"."id" AS t1_r0, "comments"."body" AS t1_r1, "comments"."post_id" AS t1_r2, "comments"."created_at" AS t1_r3, "comments"."updated_at" AS t1_r4 FROM "posts" LEFT OUTER JOIN "comments" ON "comments"."post_id" = "posts"."id" WHERE (comments.post_id = 1 )
Comentário 1 do Post 1
```

### Pré-carregando mais de uma associação

Apesar de nos exemplos anteriores termos pré-carregado apenas uma associação (`comments`), todos os métodos aqui citados (`preload`, `eager_load` e `includes`) aceitam, na verdade, uma *hash*. Então, supondo que além de comentários, tenhamos também uma outra associação em **Post** chamada **Tags**, podemos fazer:

```ruby
Post.includes(:comments, :tags)
```

Supondo, ainda, que cada comentário possua um usuário:

```ruby
Post.includes(comments: :user, :tags)
```

Ou, que, além de um usuário por comentário, também haja uma associação de comentário com **Tags**:

```ruby
Post.includes(comments: [:user, :tags], :tags)
```

### Um caso especial

Até agora vimos como utilizar pré-carregamento para evitar *queries* N+1. Porém, há um caso bem específico e relativamente comum que precisa de uma atenção especial.

Imagine o seguinte cenário: Precisamos listar todos os posts, e para cada post, a quantidade de comentários que o mesmo possui.

Se utilizarmos do que já vimos anteriormente, poderíamos fazer:

```ruby
irb(main):006> Post.includes(:comments).all.each { |post| puts post.comments.count };0
  Post Load (1.4ms)  SELECT "posts".* FROM "posts"
  Comment Load (1.0ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" IN ($1, $2, $3, $4, $5)  [["post_id", 1], ["post_id", 2], ["post_id", 3], ["post_id", 4], ["post_id", 5]]
  Comment Count (4.6ms)  SELECT COUNT(*) FROM "comments" WHERE "comments"."post_id" = $1  [["post_id", 1]]
1
  Comment Count (0.3ms)  SELECT COUNT(*) FROM "comments" WHERE "comments"."post_id" = $1  [["post_id", 2]]
1
  Comment Count (0.3ms)  SELECT COUNT(*) FROM "comments" WHERE "comments"."post_id" = $1  [["post_id", 3]]
1
  Comment Count (0.3ms)  SELECT COUNT(*) FROM "comments" WHERE "comments"."post_id" = $1  [["post_id", 4]]
1
  Comment Count (0.3ms)  SELECT COUNT(*) FROM "comments" WHERE "comments"."post_id" = $1  [["post_id", 5]]
1
=> 0
```

E vimos que, mesmo utilizando o método `includes`, ainda assim é feita uma *query* de contagem para cada Post. Isto ocorre pois o método `count` sempre dispara uma *query*, simplesmente ignorando se os resultados já estão carregados ou não.

Para contornar isto, podemos utilizar o método `size`, que funciona de uma forma parecida com o método `includes`. Caso as associações já estejam carregadas, `size` irá chamar o método `length`, que irá contar o número de registros carregados em memória do modelo. Caso não, irá chamar o `count`, que como dito acima, sempre irá disparar uma *query* do tipo `COUNT` no banco de dados.

Apesar de ser possível corrigir a *query* N+1 mudando o método de `count` para `size`, considerando uma situação onde não seria necessário acessar a associação para nada, apenas para contagem, pré-carregar essas associações em memória e contá-las pode não ser o jeito mais eficiente.

Para casos onde se precisa manter uma contagem de uma associação, o jeito mais recomendável é utilizar o `counter_cache`, onde será criado uma coluna específica na tabela apenas para guardar a contagem de uma determinada associação.

```ruby
# app/models/Comment

class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
end
```

Para utilizar o código acima será preciso criar mais um campo em Post, com o nome `comments_count`. A partir daí, o Rails automaticamente irá atualizar o contador quando um comentário for adicionado ou removido.

Para os comentários que já existem, basta rodar no *console* (ou na própria *migration*):

```ruby
Post.find_each do |post|
  Post.reset_counters(post.id, :comments)
end
```

## Identificando Queries N+1

Até agora vimos o que são _queries_ N+1 e como resolvê-las. E para identificá-las? Se você reparar nos exemplos anteriores, por definição, _queries_ N+1 se manifestam em _loops_.

Logo, toda parte da sua base de código em que uma relação de objetos do ActiveRecord está sendo iterada (com `each`, `each_slice`, `find_each`, `map`, etc), há ali uma possibilidade de _queries_ N+1.

Porém, para encontrá-las, o melhor a se fazer é utilizar alguma ferramenta que as detectem. Ficar procurando no seu código, salvo algumas exceções, além de improdutivo, pode levar à muitas conclusões erradas.

### strict_loading

O modo `strict_loading` foi adicionado no Rails 6.1, justamente para evitar o _lazy loading_ em associações, e com isso mitigar _queries_ N+1. Quando o modo `strict_loading` é ativado, as associações precisarão ser pré-carregadas, caso contrário será lançada uma exceção ou um _log_, dependendo da configuração.

O modo pode ser ativado em:

- Registros individuais
- Modelos
- Associações
- Aplicação

#### strict_loading em regitros

Neste modo, basta encadearmos o método `strict_loading` quando estivermos fazendo uma _query_:

```ruby
post = Post.strict_loading.find(1)
```

Se tentarmos acessar `post`, será possível, porém, se tentarmos acessar `comments`:

```ruby
irb(main):005> post.comments
An error occurred when inspecting the object: #<ActiveRecord::StrictLoadingViolationError: `Post` is marked for strict_loading. The Comment association named `:comments` cannot be lazily loaded.>
```

#### strict_loading em modelos

Para ativar o modo `strict_loading` em modelos, devemos colocar `self.strict_loading_by_default = true` no modelo:

```ruby
# models/post.rb

class Post < ApplicationRecord
  self.strict_loading_by_default = true

  has_many :comments
end
```

Neste modo, basta fazermos _queries_ normalmente:

```ruby
irb(main):002> post = Post.find(1)
  Post Load (1.3ms)  SELECT "posts".* FROM "posts" WHERE "posts"."id" = $1 LIMIT $2  [["id", 1], ["LIMIT", 1]]
=>
#<Post:0x00007f1a5d0f9370
...

irb(main):003> post.comments
An error occurred when inspecting the object: #<ActiveRecord::StrictLoadingViolationError: `Post` is marked for strict_loading. The Comment association named `:comments` cannot be lazily loaded.>
```

#### strict_loading em associações

Se sua preocupação é apenas com uma associação em específico, basta adicionar `strict_loading: true` na associação:

```ruby
# models/post.rb

class Post < ApplicationRecord
  has_many :comments, strict_loading: true
end
```

E, igualmente ao método anterior, basta fazermos as _queries_ normalmente:

```ruby
irb(main):005> post = Post.find(1)
  Post Load (0.4ms)  SELECT "posts".* FROM "posts" WHERE "posts"."id" = $1 LIMIT $2  [["id", 1], ["LIMIT", 1]]
=>
#<Post:0x00007f1a5d2a16f0
...
irb(main):006> post.comments
An error occurred when inspecting the object: #<ActiveRecord::StrictLoadingViolationError: `Post` is marked for strict_loading. The Comment association named `:comments` cannot be lazily loaded.>
```

#### strict_loading globalmente

Para ativar o modo `strict_loading` na aplicação inteira, basta colocarmos em algum dos arquivos de configuração por ambiente do Rails:

```ruby
# config/environments/test.rb | config/environments/development.rb | config/environments/production.rb

config.active_record.strict_loading_by_default = true
```

#### Mudando output do strict_loading para logs

Se ativarmos o `strict_loading` em qualquer modo citado acima, por padrão, ao tentar carregar uma associação afetada sem antes pré-carregar, será lançada uma exceção. Já imaginou ativar isso em produção e ir consertando conforme os usuários reclamam que "algo parou de funcionar"? Talvez, por este motivo, a equipe do Rails decidiu colocar a opção de trocarmos a exceção pela geração de um _log_:

```ruby
# config/application.rb

config.active_record.action_on_strict_loading_violation = :log
```

Diferente da ativação do `strict_loading` em si, esta é um configuração global, então não é possível deixar para mostrar _logs_ em ambiente de produção e, para gerar exceções em ambiente de teste, por exemplo.

O _log_ gerado se parecerá com:

> `Post` is marked for strict_loading. The Comment association named `:comments` cannot be lazily loaded.`

E, como vemos, não é gerado mais uma exceção:

```ruby
irb(main):001> post = Post.find(1)
  Post Load (1.0ms)  SELECT "posts".* FROM "posts" WHERE "posts"."id" = $1 LIMIT $2  [["id", 1], ["LIMIT", 1]]
=>
#<Post:0x00007fea3e355590
...
irb(main):002> post.comments
`Post` is marked for strict_loading. The Comment association named `:comments` cannot be lazily loaded.
  Comment Load (1.6ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = $1  [["post_id", 1]]
=>
[#<Comment:0x00007fea3ffcbe40
  id: 1,
  body: "Comentário 1 do Post 1",
  post_id: 1,
  created_at: Fri, 07 Jun 2024 01:51:15.385283000 UTC +00:00,
  updated_at: Fri, 07 Jun 2024 01:51:15.385283000 UTC +00:00>]
```

Se o *log* gerado não te ajuda, uma vez que provavelmente vai ficar perdido numa infinidade de *logs* sobre outras coisas, quando estamos no modo "*logs*", o Rails emite um evento toda vez que uma associação sofre um _lazy loading_. Evento este, que, você pode se inscrever e fazer o que quiser com ele, inclusive enviá-lo para algum outro serviço de *logs* ou erros, tais como Airbrake ou Honeybadger.

```ruby
# config/initializers/strict_loading_violation.rb

ActiveSupport::Notifications.subscribe("strict_loading_violation.active_record") do |name, started, finished, unique_id, data|
  model = data.fetch(:owner)
  ref = data.fetch(:reflection)

  Airbrake.notify("strict_loading_violation", model: model.name, association: ref.name)
end
```

### Gems

O _strict_loading_ só está disponível nas versões 6.1 e acima do Rails e entretanto, _queries_ N+1 são um problema desde praticamente a primeira versão do ActiveRecord. Para as aplicações que estão abaixo da versão 6.1, felizmente há _gems_ para ajudar.

Apesar de apenas falarmos de algumas *gems*, há diversas outras que podem também ajudar com a identificação de *queries* N+1, tais como [rack-mini-profiler](https://github.com/MiniProfiler/rack-mini-profiler), [N + 1 Control](https://github.com/palkan/n_plus_one_control), entre outros.

#### [Bullet](https://github.com/flyerhzm/bullet)

A _gem_ Bullet é uma das mais antigas das que se propõe a tentar resolver este problema de *queries* N+1. Historicamente era utilizada apenas em ambiente de testes, porém, hoje, já é possível desativar o levantamento de exceções deixando apenas _logs_ sendo gerados.

O problema ao rodar este tipo de ferramenta apenas em testes/desenvolvimento é que, para elas efetivamente identificarem as *queries* N+1, tais *queries* precisam acontecer primeiro, o que pressupõe testes/base de dados de desenvolvimento bem preenchida, com todas associações tendo vários registros, o que não é tão comum assim.

Outras _features_ dessa _gem_ são, por exemplo, _pop-ups_ no _browser_ e detecção de pré-carregamento desnecessário.

O grande problema com a _gem_ Bullet, segundo os próprios usuários, é a alta quantidade de falsos positivos/negativos que ela gera. Exatamente por conta disso a _gem_ [Prosopite](https://github.com/charkost/prosopite) foi criada.

#### [Prosopite](https://github.com/charkost/prosopite)

Criada com o propósito de detectar _queries_ N+1 sem falso positivos/negativos, a _gem_ Prosopite tem exatamente o mesmo propósito da _gem_ discutida anteriormente, podendo também ser configurada para lançar ou não exceções. Ela pode, portanto, ser utilizada tanto em produção, quanto em desenvolvimento/testes.

#### [Goldiloader](https://github.com/salsify/goldiloader)

Até agora as *gems* que vimos tinham como objetivo detectar *queries* N+1. A *gem* Goldiloader visa resolvê-las automaticamente antes que elas ocorram. Utilizando esta *gem*, toda vez que você tenta acessar uma associação, ela é pré-carregada automaticamente. Também é possível desativar este pré-carregamento automático manualmente, caso necessário.

Apesar de parecer que irá resolver todos os problemas, é importante reconhecer que a *gem* possui [limitações](https://github.com/salsify/goldiloader?tab=readme-ov-file#limitations).

### ScoutAPM

O [ScoutAPM](https://www.scoutapm.com), como o próprio nome diz, é um APM (*Application Performance Monitoring*), onde é possível analisar o desempenho de sua aplicação por diversos ângulos, tais como uso de memória por requisição, *queries* lentas, e *queries* N+1.

O lado negativo deste serviço é que ele é grátis somente - *no momento em que este post é escrito* - se sua aplicação gerar menos que 300 mil *transactions* por mês. Se sua aplicação gerar mais que isso, e, você não quiser ou não tiver como pagar um plano, uma opção é diminuir o número de transações enviadas:

```ruby
# app/controllers/application_controller.rb

class ApplicationController < ActionController::API
  before_action :sample_requests_for_scout

  private

  def sample_requests_for_scout
    sample_rate = 0.5

    ScoutApm::Transaction.ignore! if rand > sample_rate
  end
end
```

O código acima, por exemplo, envia 50% das transações que normalmente seriam enviadas para o ScoutAPM. E, obviamente, apenas 50% das requisições estarão disponíveis no ScoutAPM para serem analisadas.

## Analisando Queries

Uma vez que você encontrou as *queries* N+1 que estão deixando sua aplicação Rails de joelhos, nem sempre a solução será fácil de ser enxergada. Ou, ainda que seja, não faz mal uma prova real.

Para analisar tais trechos, é imprescindível que consigamos ver o problema de forma empírica, então testar no _console_ do Rails é uma ótima opção. Para conseguir "ver" as queries sendo rodadas, certifique-se que os _logs_ do ActiveRecord estejam ativados:

```ruby
ActiveRecord::Base.logger = Logger.new STDOUT
```

### APIs REST

Um dos modos possíveis que o Rails pode ser utilizado é o API. Neste modo há um local nem tão óbvio que é, pela minha experiência, uma fonte inesgotável de _queries_ N+1: **Serializers**.

Então, supondo que uma das soluções apresentadas aqui tenha apontado que temos uma *query* N+1 no *endpoint* de listagem de Posts.

Ao checar o controller e toda estrutura envolvida para este *endpoint*:

```ruby
# app/controllers/posts_controller.rb

class PostsController < ApplicationController
  def index
    posts = PostsServices::Index.new.last_5

    render json: posts, each_serializer: PostSerializer
  end
end
```

```ruby
# app/services/posts_services/index.rb

module PostsServices
  class Index
    def last_5
      Post.last(5)
    end
  end
end
```

```ruby
# app/serializers/post_serializer.rb

class PostSerializer < ActiveModel::Serializer
  attributes :id, :title, :comments
end
```

Considerando a hipótese acima, para ver as *queries* N+1 acontecendo, basta que executemos, no *console*, exatamente como está no *controller* (considerando que haja pelo menos um comentário por post):

```ruby
irb(main):004> posts = PostsServices::Index.new.last_5
  Post Load (0.2ms)  SELECT "posts".* FROM "posts" ORDER BY "posts"."id" DESC LIMIT $1  [["LIMIT", 5]]
=>
[#<Post:0x00007f1c8bcec6c0
...
irb(main):005> ApplicationController.render json: posts, each_serializer: PostSerializer
`Post` is marked for strict_loading. The Comment association named `:comments` cannot be lazily loaded.
  Comment Load (1.7ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = $1  [["post_id", 1]]
`Post` is marked for strict_loading. The Comment association named `:comments` cannot be lazily loaded.
  Comment Load (0.4ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = $1  [["post_id", 2]]
`Post` is marked for strict_loading. The Comment association named `:comments` cannot be lazily loaded.
  Comment Load (0.3ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = $1  [["post_id", 3]]
`Post` is marked for strict_loading. The Comment association named `:comments` cannot be lazily loaded.
  Comment Load (0.2ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = $1  [["post_id", 4]]
`Post` is marked for strict_loading. The Comment association named `:comments` cannot be lazily loaded.
  Comment Load (0.2ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = $1  [["post_id", 5]]
Rendered ActiveModel::Serializer::CollectionSerializer with ActiveModelSerializers::Adapter::Attributes (31.72ms)
=> "[{\"id\":1,\"title\":\"Post 1\",\"comments\":[{\"id\":1,\"body\":\"Comentário 1 do Post 1\",\"post_id\":1,\"created_at\":\"2024-06-07T01:51:15.385Z\",\"updated_at\":\"2024-06-07T01:51:15.385Z\"}]},{\"id\":2,\"title\":\"Post 2\",\"comments\":[{\"id\":6,\"body\":\"Comentário 1 do Post 2\",\"post_id\":2,\"created_at\":\"2024-06-07T02:43:11.069Z\",\"updated_at\":\"2024-06-07T02:43:11.069Z\"}]},{\"id\":3,\"title\":\"Post 3\",\"comments\":[{\"id\":7,\"body\":\"Comentário 1 do Post 3\",\"post_id\":3,\"created_at\":\"2024-06-07T02:43:17.323Z\",\"updated_at\":\"2024-06-07T02:43:17.323Z\"}]},{\"id\":4,\"title\":\"Post 4\",\"comments\":[{\"id\":8,\"body\":\"Comentário 1 do Post 4\",\"post_id\":4,\"created_at\":\"2024-06-07T02:43:21.111Z\",\"updated_at\":\"2024-06-07T02:43:21.111Z\"}]},{\"id\":5,\"title\":\"Post 5\",\"comments\":[{\"id\":9,\"body\":\"Comentário 1 do Post 5\",\"post_id\":5,\"created_at\":\"2024-06-07T02:43:24.323Z\",\"updated_at\":\"2024-06-07T02:43:24.323Z\"}]}]"
```

Após colocar o pré-carregamento, basta dar o comando `reload!` no *console* do Rails e testar novamente para ver se a *query* N+1 foi corrigida.

## Conclusão

Apesar de ser um problema sério que acarreta em diversos problemas de desempenho, é tão antigo quanto a existência dos ORMs, e por conta disso é bem conhecido, bem documentado e com diversas opções de detecção, tanto oficiais, quanto por *gems* da comunidade.