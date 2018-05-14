---
layout: post
title:  "How to Use Sidekiq"
date:   2018-05-14 16:10:33 +0900
categories: sidekiq
---

# 如何使用sidekiq
## 什么是sidekiq
提供了异步处理功能的[gem](https://github.com/mperham/sidekiq)<br/>
这里简单了归纳了一下如何在rails的项目里面使用sidekiq

## 准备工作
异步处理的数据放在redis里面，所以需要安装redis<br/>
如果用的是mac的话，可以用```brew install redis```安装

## 添加Gem

在Gemfile里面添加，然后```bundle install --path vender/bundle```进行安装

```ruby:Gemfile
gem 'sidekiq'
```

## 设置
创建一个文件 config/initializers/sidekiq.rb

```ruby
require 'sidekiq/web'

Sidekiq.configure_server do |config|
  config.redis = { url: "#{Settings.redis.url}/#{Settings.redis.sidekiq_db}", namespace: Settings.redis.sidekiq_namespace }
  config.server_middleware do |chain|
    chain.add Sidekiq::Middleware::Server::RetryJobs, max_retries: 4
  end
end
Sidekiq.configure_client do |config|
  config.redis = { url: "#{Settings.redis.url}/#{Settings.redis.sidekiq_db}", namespace: Settings.redis.sidekiq_namespace }
end
```
开发环境和实际的服务器环境的redis的设置不同，所以放在settings里面
```yaml
redis:
  url: <%= ENV['REDIS_URL'] || 'redis://127.0.0.1:6379' %>
  db: '0'
  sidekiq_db: '1'
  sidekiq_namespace: sidekiq<%= ENV['RAILS_ENV'] %>
```

## 发送邮件
在application.rb里面添加以下代码
```ruby
  config.active_job.queue_adapter       = :sidekiq
```

然后发送邮件的时候使用deliver_later就可以异步发送

```ruby
UserMailer.send_mail(User.find(1)).deliver_later
```

## Worker
进行异步处理的worker类都放在app/workers文件夹下面

比如要对用户数据进行排名
app/workers/use_ranking_worker.rb
```ruby
class UserRankingWorker
  include Sidekiq::Worker
  sidekiq_options queue: :event, retry: 3

  def perform(user_id)
    user = User.find_by(user_id)
    return if user.blank?
    ## do something
  end
end
```

## 把人物加入Queue(队列)
使用perform_async方法来加入队列

```ruby
class UserService
  def ranking(user)
    UserRankingWorker.perform_async user.id
  end
end
```

也可以使用perform_in在指定时间后执行

```ruby
class UserService
  def ranking(user)
    UserRankingWorker.perform_in 1.hour, @event.id
  end
end
```

## sidekiq起動
### 设置
config/sidekiq.yml
```ruby
# thread数
:concurrency: 35
:pidfile: ./tmp/pids/sidekiq.pid
:logfile: ./log/sidekiq.log
:daemon: true
# 对需要立刻处理的任务指定优先级
:queues:
  - [mailers, 100]
  - [events, 1]
```

### 启动sidekiq
```
bundle exec sidekiq -C config/sidekiq.yml -d
```

## 发布
需要注意的是，新的代码更新后，sidekiq必须重新启动才可以<br/>
现在的项目用的是capistrano来进行发布,添加[capistrano-sidekiq](https://github.com/seuros/capistrano-sidekiq)这个GEM后，在config/deploy.rb添加一下代码后，可以在每次发布后重新启动
config/deploy.rb
```ruby
set :sidekiq_role, :worker
set :sidekiq_config, "#{current_path}/config/sidekiq.yml"
SSHKit.config.command_map[:sidekiq] = 'bundle exec sidekiq'
SSHKit.config.command_map[:sidekiqctl] = "bundle exec sidekiqctl"
```
为了不影响web服务器，所以新建了一个worker服务器，专门用来运行sidekiq，可以根据实际情况进行调整

## 管理画面
sidekiq自带了一个管理画面，按照下面的方法可以很容易的添加<br/>
可以看到正在进行的任务，失败的任务，成功的任务<br/>
失败的任务还可以选择以后从画面上重启<br/>

添加Gem
```ruby:Gemfile
gem 'sinatra', require: false
```

在routes.rb追加routing
config/routes.rb
```ruby
require 'sidekiq/web'

MyApp::Application.routes.draw do
  authenticate :operator, lambda { |o| o.developer? } do
    mount Sidekiq::Web => '/sidekiq'
  end
end
```
因为这个画面不能让任何人都看到，所以设置成在管理画面里面并且是开发人员才可以看到
然后/sidekiq就可以看到了

## 总结
在提高服务器的应答速度的时候，可以进行异步处理的部分都可以用sidekiq来处理，这样可以有效提高服务器应答速度<br/>
发送邮件之类的，由于网络原因或者邮件服务器原因一时无法发送的时候可以重新执行任务，非常简单实用

