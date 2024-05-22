+++
title = "Uma \"pegadinha\" nas transactions do Rails"
date = 2024-05-21
draft = false

[taxonomies]
categories = ["rails"]
tags = ["ruby", "rails", "active record", "transactions", "callbacks"]

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

Cuidado ao utilizar callbacks com transactions no Rails

<!-- more -->

# O problema

Recentemente, tivemos um problema em que um serviço estava gerando registros inválidos no banco de dados. A reclamação era que, mesmo apresentando erro, tarefas estavam sendo parcialmente atualizadas no banco de dados.

O problema foi fácil de ser identificado. Se analisarmos o seguinte esboço de código do serviço, vemos que um mesmo registro é atualizado mais de uma vez:

```ruby
module TaskServices
  class Update
    def execute
      update_task
      update_meeting if task.meeting?
    end

    def update_task
      # ...
      @task.save!
      # ...
    end

    def update_meeting
      # ...
      @task.save!
      # ...
    end
  end
end
```

# A solução

O erro ocorria no segundo método, `update_meeting`, porém um `@task.save!` já tinha sido chamado anteriormente, realizando `commit` no banco de dados. Para resolver, bastou "embrulhar" os métodos em uma _transaction_:

```ruby
def execute
  ActiveRecord::Base.transaction do
    update_task
    update_meeting if task.meeting?
  end
end
```

Quando colocamos diversas operações dentro de uma _transaction_, o _commit_ no banco de dados só é feito ao final da _transaction_ e apenas se nada de errado ocorrer. Desta forma, se por qualquer motivo o segundo `@task.save!` falhar, o primeiro **NÃO** será salvo no banco. Problema resolvido!

# O problema da solução

Alguns dias depois chega uma nova reclamação de que eventos de tarefas do tipo _meeting_ não estão sendo gerados...

Se analisarmos o código que deveria gerar o evento:

```ruby
class Task < ApplicationRecord
  # ...
  after_commit :register_event, if: :saved_change_to_done?
  # ...

  def register_event
    Event.create # ...
  end
```

Acontece que o atributo `done` é alterado no primeiro método, `update_task`, porém o método `register_event` simplesmente não estava mais sendo chamado após a implementaçao da _transaction_.

Testando aqui e acolá, em algum momento acabei removendo o `if: :saved_change_to_done?` e o método foi chamado.

Aí que a ficha caiu: Atributos do ActiveModel::Dirty são resetados cada vez que um método que altera um modelo é chamado (`save!` no nosso caso), independentemente se está dentro de uma _transaction_ ou não.

Então, o que ocorreu foi que ao chamar o primeiro `save!`, os atributos do ActiveModel::Dirty foram apagados sem que o `after_commit` fosse chamado (uma vez que não houve _commit_ ainda). Após o segundo `save!` ser finalizado, a _transaction_ é concluída, gerando um _commit_ no banco de dados e assim os `after_commit` são chamados, porém, como os atributos do ActiveModel::Dirty do primeiro `save!` foram resetados, apenas os atributos do segundo `save!` é que estarão disponíveis para consulta, logo, `saved_change_to_done?` retorna `false`, uma vez que não houve mudança no atributo `done` no último `save!`.

# A solução da solução

Para resolver este problema, há basicamente duas opções:

- Não alterar um mesmo registro mais de uma vez em uma _transaction_
- Utilizar a _gem_ [ar_transaction_changes](https://github.com/dylanahsmith/ar_transaction_changes)

Apesar da primeira opção parecer ser óbvia e até uma "boa prática", alterar um mesmo registro várias vezes em um serviço pode ser comum, principalmente se você segue o princípio _DRY_, pois você pode estar reutilizando outros serviços dentro deste (e estes serviços alterarem este mesmo registro).

A ressalva quanto a segunda opção fica aos possíveis conflitos que esta _gem_ pode ter com outras _gems_ (ler o _README_) e, que, dependendo do tamanho e idade da sua base de código, pode ser que mais alguém se deparou com este problema e simplesmente contornou ele, sem necessariamente ter entendido o que aconteceu. Se for este o caso, adicionar a _gem_ pode fazer com que mais _callbacks_ sejam chamados de forma inesperada, podendo ocasionar em novos _bugs_.