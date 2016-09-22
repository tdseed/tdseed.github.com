---
layout: post
title:  "微信应用开发"
date:   2016-09-12 10:10:09
categories: technology
author: tank
---

如果单纯只用于微信应用，不考虑支付宝应用的话，可以借鉴有些gem的设计，要支持支付宝应用，需要另外处理

我的开发环境
mac、sublime

## 使用的技术栈：
Rails5.0.0.1 (5.0.0.1是最新发布的稳定版本，我从rails3.2开始使用，使用最多的是rails4.2.5)
Ruby 2.3.1 (最新发布的稳定版本，我看Ruby 2.4.0-preview1，暂时不用，之后Ruby3.0据说还要到2020年，功能会更加强大)
PostgreSQL (The world's most advanced open source database, 之前很多项目从mysql迁移到PostgreSQL，它吸取了mysql、mongodb等数据库的
长处，整合在一起，比如json的支持，权限控制，热部署，主从复制（比如流复制）更先进，运维成本跟mysql差不多当你熟悉了之后）

### 服务器
为了方便我测试服务器用了Heroku，(https://www.heroku.com/)
正式的可以使用阿里云、熟悉ubuntu、centos等，rails可以用capistrano部署，capistrano应该也是支持java应用的，(https://github.com/capistrano/capistrano)
capistrano需要在rails代码里面配置好，然后即可一个命令实现热部署
rails应用服务器用的puma (http://puma.io/)，这个相对应java的Tomcat，puma多线程架构，热启动
关于puma，我以前的技术总监写了完整的源码架构解析，非常酷https://ruby-china.org/topics/24378

### 使用的gem:

```ruby
  source 'https://rubygems.org'

  # Bundle edge Rails instead: gem 'rails', github: 'rails/rails'
  gem 'rails', '~> 5.0.0', '>= 5.0.0.1'
  # Use sqlite3 as the database for Active Record
  # gem 'sqlite3'
  # Use Puma as the app server
  gem 'puma', '~> 3.0'
  # Use SCSS for stylesheets
  gem 'sass-rails', '~> 5.0'
  # Use Uglifier as compressor for JavaScript assets
  gem 'uglifier', '>= 1.3.0'
  # Use CoffeeScript for .coffee assets and views
  gem 'coffee-rails', '~> 4.2'
  # See https://github.com/rails/execjs#readme for more supported runtimes
  # gem 'therubyracer', platforms: :ruby

  # Use jquery as the JavaScript library
  gem 'jquery-rails'
  # Turbolinks makes navigating your web application faster. Read more: https://github.com/turbolinks/turbolinks
  gem 'turbolinks', '~> 5'
  # Build JSON APIs with ease. Read more: https://github.com/rails/jbuilder
  gem 'jbuilder', '~> 2.5'
  # Use Redis adapter to run Action Cable in production
  # gem 'redis', '~> 3.0'
  # Use ActiveModel has_secure_password
  # gem 'bcrypt', '~> 3.1.7'

  # Use Capistrano for deployment
  # gem 'capistrano-rails', group: :development

  group :development, :test do
    # Call 'byebug' anywhere in the code to stop execution and get a debugger console
    gem 'byebug', platform: :mri
  end

  group :development do
    # Access an IRB console on exception pages or by using <%= console %> anywhere in the code.
    gem 'web-console'
    gem 'listen', '~> 3.0.5'
    # Spring speeds up development by keeping your application running in the background. Read more: https://github.com/rails/spring
    gem 'spring'
    gem 'spring-watcher-listen', '~> 2.0.0'
  end

  # Windows does not include zoneinfo files, so bundle the tzinfo-data gem
  gem 'tzinfo-data', platforms: [:mingw, :mswin, :x64_mingw, :jruby]

  gem 'pry', group: :development

  gem 'figaro'
  gem 'devise'
  gem 'wechat'
  gem 'wx_pay'
  gem 'weui-rails'
  gem 'omniauth-wechat-oauth2'
  gem 'pg'
```
## 分析

重点跟微信有关的wechat、wx_pay、weui-rails、omniauth-wechat-oauth2
前端使用weui (https://github.com/weui/weui)
通过修改可以获取微信用户信息

查看了我公司restful风格的api，我需要验签一下，确认一下接口可用性，然后把这些api封装成方法，
目前wechat，开启wechat-api，开发的应用使用了JS_SDK, 然后在微信公众号开发进行相关设置

本地端口代理到公网我使用了localtunnel （http://localtunnel.me/）
这样就可以本地调试了，不过这个微信会多个页面说你的页面不安全，多个点击步骤，我尝试后觉得就是这个不稳定

为了方便快速部署到测试服务器，我使用了Heroku, 演示地址使用了我的微信开发测试号

## 微信应用号

微信应用号在9月22号左右出来，应该是基于js_sdk的架构，还有自己定义了一套xml写界面，类似app store的模式，这些可以留意最新的新闻

## 我搭建的测试号

![Alt text](http://7xjjp8.com1.z0.glb.clouddn.com/0.jpg 2016-09-12 上午12.36.50.png "weixin")
