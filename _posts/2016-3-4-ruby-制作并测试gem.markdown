---
layout: post
title:  制作gem并测试
date:   2016-3-4 17:49:00
categories: linux
---
查看bundle的版本
junjunlocal:projects Admin$ bundle -v
Bundler version 1.10.5

使用bundle gem 生成一个gem项目
junjunlocal:projects Admin$ bundle gem rc-client
Creating gem 'rc-client'...
Code of conduct enabled in config
MIT License enabled in config
      create  rc-client/Gemfile
      create  rc-client/.gitignore
      create  rc-client/lib/rc/client.rb
      create  rc-client/lib/rc/client/version.rb
      create  rc-client/rc-client.gemspec
      create  rc-client/Rakefile
      create  rc-client/README.md
      create  rc-client/bin/console
      create  rc-client/bin/setup
      create  rc-client/CODE_OF_CONDUCT.md
      create  rc-client/LICENSE.txt
      create  rc-client/.travis.yml
Initializing git repo in /User/junjun/projects/rc-client

使用RSpec进行测试,在根目录下创建一个spec目录
junjunlocal:rc-client Admin$ mkdir spec
junjunlocal:rc-client Admin$ ls
CODE_OF_CONDUCT.md LICENSE.txt        Rakefile           lib                spec
Gemfile            README.md          bin                rc-client.gemspec

在rc-client.gemspec中加入依赖 spec.add_development_dependency "rspec", "~> 3.2"
Gem::Specification.new do |spec|
  ...
  ...
  spec.add_development_dependency "bundler", "~> 1.10"
  spec.add_development_dependency "rake", "~> 10.0"
  spec.add_development_dependency "rspec", "~> 3.2"
end

