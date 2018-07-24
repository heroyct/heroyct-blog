---
layout: post
title:  "sidekiqの使い方"
date:   2018-05-14 16:10:33 +0900
categories: sidekiq
---

# sidekiqの使い方
## sidekiqとは
非同期処理の仕組みを提供するgem、[sidekiq](https://github.com/mperham/sidekiq)あたりを見れがわかりますが、
Railsで使う場合について簡単にまとめてみる

## 準備
バックエンドにredisが必要です。
MACでローカルで試すなら、brew install redisでインストールできる
本番はredisサーバーを立てる必要あります

## インストール

Gemfileに書いてbundle installする

```ruby:Gemfile
gem 'sidekiq'
```

## 設定

initializersに起動時の設定を書きます。

```config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
    config.redis = { url: 'redis://localhost:6379', namespace: 'sidekiq' }
    config.server_middleware do |chain|
      chain.add Sidekiq::Middleware::Server::RetryJobs, max_retries: 4
    end
end
Sidekiq.configure_client do |config|
  config.redis = { url: 'redis://localhost:6379', namespace: 'sidekiq' }
end
```


## Worker

非同期に実行するworkerをapp/workers以下に配置します。
workerにperformというメソッドを作成し、その中に実行されるべきロジックを書きます。
実際のロジックはモデルに記述した方がテストしやすいと思います。
引数にモデルのインスタンスを取ることもできますが、いちいちシリアライズされるためパフォーマンスが低下します。なるべくidを渡してWorker内でロードするようにした方が賢明でしょう。
sidekiq_optionsのqueueにはnamespaceを指定します。
未指定だとdefaultというnamespaceになります。
sidekiqを起動する際に必要になるので覚えておいて下さい。

```app/workers/event_worker.rb
class EventWorker
  include Sidekiq::Worker
  sidekiq_options queue: :event

  def perform(id)
    @event = Event.find(id)
    @event.calculate_rank!
  end
end
```

## キューに入れる

perform_asyncを実行するとworkerがキューに入り、非同期でperformメソッドが実行されます。
通常、コントローラから呼び出されると思います。

```app/controllers/events_controller.rb
class EventsController
  def ranking
    EventWorker.perform_async @event.id
  end
end
```

perform_asyncの代わりにperform_inを使うと指定時間だけ待ってから実行されます。

```app/controllers/events_controller.rb
class EventsController
  def ranking
    EventWorker.perform_in 1.hour, @event.id
  end
end
```

## sidekiq起動

キューに入れただけではジョブは実行されません。
とりあえずCUIで実行するには以下のようにします。

```
bundle exec sidekiq -q default event
```

namespaceが複数ある場合は全て指定します。
指定しないとdefaultのキューのみが実行されます。
しばらく待ってキューが実行され、コンソールにログが流れれば成功です。

## 起動設定

毎回上記のオプションを入力するのは面倒なので、設定ファイルを作成します。

```config/sidekiq.yml
:verbose: false
:pidfile: ./tmp/pids/sidekiq.pid
:logfile: ./log/sidekiq.log
:concurrency:  25
:queues:
  - default
  - event
```

設定ファイルを読み込んで起動する場合、以下のようにします。

```
bundle exec sidekiq -C config/sidekiq.yml
```


## デプロイ

sidekiqは一度起動すれば（落ちるまで）動き続けますが、注意するのはデプロイ時です。sidekiqがロードしているコードは更新されないので、再起動を忘れると思わぬバグが発生する可能性があります。
capistranoを使っている場合、デプロイと同時に起動・再起動ができます。

```config/deploy.rb
require 'sidekiq/capistrano'

set :sidekiq_role, :web
```

roleはsidekiqを走らせるroleです。複数台でsidekiqを実行すると自動的にバランシングされるので、特別な理由がなければrailsを走らせているroleを指定するのが良いでしょう。
設定ファイルはデフォルトで上記のconfig/sidekiq.ymlが使われます。
これでcap deployを実行すれば自動的にsidekiqが起動・再起動します。

## リトライ・エラー処理

sidekiqはperformメソッドが例外を返すと自動的にリトライします。
処理に失敗した場合にリトライさせたいなら、raiseすればいいわけです。
retryに数字を指定すると、その回数だけリトライして諦めます。

```app/workers/event_worker.rb
  sidekiq_options queue: :event, retry: 5
```

デフォルトではリトライはリトライ回数の4乗＋15秒間隔で行われます。
即ち15,16,31,96,271…秒の間隔を置きます。
変更する場合はsidekiq_retry_inを定義します。
詳しくは[wiki](https://github.com/mperham/sidekiq/wiki/Error-Handling)を参照して下さい。

リトライさせたくない場合、retryに数字ではなくfalseを指定します。

```app/workers/event_worker.rb
  sidekiq_options queue: :event, retry: false
```

## ダッシュボード

sidekiqには[こんな感じ](https://raw.github.com/mperham/sidekiq/master/examples/web-ui.png)のかっこいいダッシュボードがビルトインされています。
有効にするにはまずGemfileにsinatraを追加します。

```ruby:Gemfile
gem 'sinatra', require: false
```

次にroutes.rbにルーティングを追加します。

```config/routes.rb
require 'sidekiq/web'

MyApp::Application.routes.draw do
  mount Sidekiq::Web, at: "/sidekiq"
end
```
これで/sidekiqにアクセスすればダッシュボードが表示されます。
アクセス制限をかけたい時は以下のようにします。

```config/routes.rb
require 'sidekiq/web'

MyApp::Application.routes.draw do
  authenticate :user, lambda { |u| u.admin? } do
    mount Sidekiq::Web => '/sidekiq'
  end
end
```

## テスト

rspec-sidekiqを使うとrspecでテストが書けます。

```ruby:Gemfile
group :test do
  gem 'rspec-sidekiq'
end
```

```spec/workers/event_worker_spec.rb
require 'spec_helper'

describe EventWorker do
  let(:test_event) { create(:event) }

  before do
    subject.perform test_event.id
  end

  it "performでjobがqueueに入ること" do
    should be_processed_in(:event)
  end
end
```

processed_in?の引数にはworkerのsidekiq_optionsで指定したnamespaceが入ります。
詳しくは[rspec_sidekiq](https://github.com/philostler/rspec-sidekiq)のreadmeを参照して下さい。

## まとめ

sidekiqは非常に簡単なので是非使ってみて下さい。
個人的にはdelayed_jobやresqueよりも分かりやすいと思います。

## 追伸

rails4でも支障なく動きます。
