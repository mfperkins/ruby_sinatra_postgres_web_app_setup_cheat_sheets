<head>
<script>
  (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
  (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
  m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
  })(window,document,'script','https://www.google-analytics.com/analytics.js','ga');

  ga('create', 'UA-83875340-1', 'auto');
  ga('send', 'pageview');

</script>
</head>

# Setting up a Ruby Sinatra App on Heroku using a Postgres DB
## Step by step cheatsheet

This page is intended to provide all the command line instructions you need to get an app up and running on [Heroku](http://www.heroku.com) using the following technologies:

* Ruby
* Sinatra / Rack
* Capybara
* Postgresql
* Heroku

Setting up the project
---
1. `mkdir <project-name>` --> create a directory for your project
2. `cd <project-name>` --> switch into that directory
3. `git init` --> initialize the directory to use git
4. `gem install bundler` --> skip this step if you already have bundler
5. `bundle init` --> will create a Gemfile

  Get the Gemfile working
  ---
6. In the Gemfile add the following:

  ```ruby
  ruby '2.3.0' #or whatever version of ruby you want to use
  gem 'rspec'
  gem 'sinatra'
  gem 'rake'
  ```

7. `bundle` --> run bundle to get all the gems

  Get the Sinatra and Capybara up and running
  ---
8. `rspec-sinatra init --app MyApp ./app/app.rb` --> where MyApp == name of your app and the path points to the correct folder. Don't worry about overwriting the spec_helper.rb file ... you should also see a `config.ru` file with something like the following:

  ```ruby
  require 'rubygems'
  require File.join(File.dirname(__FILE__), 'app/app.rb')

  run MyApp
  ```

9. `rackup` and go to `localhost:9292` in your browser --> check Sinatra is working properly. You may need to use a different port - check the logs to make sure it's correct.

  Get the DB up and running
  ---
10. `psql` --> launch postgresql from the command line
11. `create database "YourDataBase_test";` --> create a database for your test environment
12. `create database "YourDataBase_development";` --> create a database for your development environment
13.  Add the following gems to your Gemfile:

  ``` ruby
  gem 'data_mapper'
  gem 'dm-postgres-adapter'
  gem 'database_cleaner'
  gem 'pg'
  ```

14. `bundle` --> to get the gems you just specified
15. `touch data_mapper_setup.rb` --> let's create a file to load up DataMapper
16. add the following to the file you just created:

  ``` ruby
  require 'data_mapper'
  require 'dm-postgres-adapter'

  # later you'll need to require the models that are using DM to connect to your DB

  DataMapper.setup(:default, ENV['DATABASE_URL'] || "postgres://localhost/YourDataBase_#{ENV['RACK_ENV']}")
  DataMapper::Logger.new($stdout, :debug)
  DataMapper.finalize
  DataMapper.auto_upgrade!
  ```

  Don't forget to point at the right DB path you just created in step 11. And add `require_relative 'data_mapper_setup'` to your `app.rb` file.

17. Make sure your are correctly set up for different environments. You should see something like `ENV['RACK_ENV'] = 'test'` in your `spec_helper.rb` file. Don't forget to require the relevant files. In your `app.rb` file add `ENV["RACK_ENV"] ||= "development"` --> that makes sure that you run the development environment by default.

18. Add `require 'database_cleaner'` to your spec_helper. And the following code after `Rspec.configure do |config|`:

  ``` ruby
  config.before(:suite) do
      DatabaseCleaner.strategy = :transaction
      DatabaseCleaner.clean_with(:truncation)
    end

    # Everything in this block runs once before each individual test
    config.before(:each) do
      DatabaseCleaner.start
    end

    # Everything in this block runs once after each individual test
    config.after(:each) do
      DatabaseCleaner.clean
    end
  ```

  This will clean out your test DB after each Rspec run ...

19. At this point you should create some simply models and verify that DataMapper is working correctly and storing things in your DB.

  Setting up with Heroku
  ---
20. `heroku create YourApp` --> to create an app on Heroku (if you leave out YourApp it will simply pick a name at random)
21. `git remote -v` --> to verify that it has added Heroku as a remote
22. `heroku addons:create heroku-postgresql:hobby-dev` --> to create a Postgresql DB on Heroku
23. `git push heroku master` --> you can also use `git push heroku your-branch:master` to push a specific branch to Heroku master
24. `heroku open` --> launches your app in a web browser
25. `heroku logs` --> to see what's going on (skip this step if it's all working!)

  Time for a Rakefile
  ---
26. `touch Rakefile` --> create one if you don't already have once
27. Add this code to it:

  ``` ruby
  require 'data_mapper'
  require './app/app.rb' # make sure you use the right path to your app

  namespace :db do
    desc "Non destructive upgrade"
    task :auto_upgrade do
      DataMapper.auto_upgrade!
      puts "Auto-upgrade complete (no data loss)"
    end


    desc "Destructive upgrade"
    task :auto_migrate do
      DataMapper.auto_migrate!
      puts "Auto-migrate complete (data was lost)"
    end
  end
  ```
28. Then run `rake db:auto_upgrade` for your local development DB and `heroku run rake db:auto_upgrade`
