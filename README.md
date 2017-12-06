# docker_heroku_rails
Demo how to build a docker rails service deployed to heroku through circleci with postgres and redis attached

- use vagrant to start environment in which to run docker
- ^ docker is NOT a VM!  (it can and frequently does run IN a VM)
- use docker image to bootstrap new rails app
- deploy to heroku
- deploy using circleci
- use a C# dotnet app instead
- use existing rails app


1. Install VirtualBox: https://www.virtualbox.org/wiki/Downloads
1. Install Vagrant: https://www.vagrantup.com/downloads.html

```
git clone git@github.com:rogusdev/docker_heroku_rails.git
cd docker_heroku_rails/
cp ../Vagrantfile ./
cp ../README.md ./

vagrant up
chmod 600 .vagrant/machines/default/virtualbox/private_key
scp -r -i .vagrant/machines/default/virtualbox/private_key -P 2222 ~/.ssh/id_rsa* ubuntu@127.0.0.1:/home/ubuntu/.ssh/
vagrant ssh


cat << EOF > .env
SECRET_KEY_BASE=597eae9051c822c955c4e44f98e75b14beb7596845678e9bf8acab5fd02e7f12some000random00number00009c62c1b
DATABASE_URL=postgres://postgres:@postgres:5432
EOF

cat << EOF > .dockerignore
.DS_Store
.dockerignore
.env
.git
.gitignore
.idea
.vagrant
.vscode
docker-compose.yml
log/*
tmp/*
EOF

cat << EOF > .gitignore
.DS_Store
.env
.idea
.vagrant
.vscode
log/*
tmp/*
.bundle
vendor/bundle
EOF

cat << EOF > Gemfile
source 'https://rubygems.org'
gem 'rails', '5.1.4'
EOF

touch Gemfile.lock

echo "web: bundle exec puma -C config/puma.rb" > Procfile


cat << EOF > app.json
{
  "name": "Docker Rails Heroku",
  "description": "An example app.json for container-deploy",
  "image": "heroku/ruby",
  "addons": [
    "heroku-postgresql"
  ]
}
EOF


cat << EOF > Dockerfile
# https://docs.docker.com/compose/rails/
# http://codingnudge.com/2017/03/17/tutorial-how-to-run-ruby-on-rails-on-docker-part-1/
# https://www.engineyard.com/blog/using-docker-for-rails
FROM ruby:2.4.2
#FROM heroku/heroku:16

RUN apt-get update -qq \\
    && apt-get install -y --no-install-recommends \\
    build-essential \\
    libpq-dev \\
    nodejs \\
    && rm -rf /var/lib/apt/lists/*

ENV RAILS_ROOT /var/www/app

RUN mkdir -p \$RAILS_ROOT/tmp/pids

WORKDIR \$RAILS_ROOT
 
COPY Gemfile Gemfile
COPY Gemfile.lock Gemfile.lock

RUN bundle install

COPY . .

CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
#CMD ["bundle", "exec", "rails", "server", "-p", "3000", "-b", "0.0.0.0"]
EOF


cat << EOF > docker-compose.yml
# https://blog.codeship.com/running-rails-development-environment-docker/
# https://nickjanetakis.com/blog/dockerize-a-rails-5-postgres-redis-sidekiq-action-cable-app-with-docker-compose
version: '3'
services:
  postgres:
    image: postgres:9.6
    ports:
      - "5432:5432"
  redis:
    image: redis:4.0.2
    ports:
     - "6379:6379"
  web:
    build: .
    volumes:
      - .:/var/www/app
    ports:
      - "3000:3000"
    env_file: ".env"
    depends_on:
      - postgres
      - redis
EOF


# https://docs.docker.com/compose/rails/

docker-compose run web rails new . --force --database=postgresql
docker-compose build
docker-compose run web rails generate controller Welcome index

sed -i "3i\  root to: 'welcome#index'" config/routes.rb

sudo sed -i "/  pool:/ a \
\  url: <%= ENV['DATABASE_URL'] %>" config/database.yml

docker-compose run web rake db:create
docker-compose run -e "RAILS_ENV=test" web rake db:create db:migrate
docker-compose run -e "RAILS_ENV=test" web rake test

docker-compose up


docker ps -a && docker images
#docker stop postgres:9.6
#docker rm postgres:9.6
#docker rmi ...

docker build -t docker-heroku-rails .

docker run --rm -it --env-file .env -p 0.0.0.0:3000:3000 docker-heroku-rails


heroku plugins:install heroku-container-registry
heroku login
heroku container:login

heroku create rogusdev-docker-heroku-rails
#heroku apps:destroy rogusdev-docker-heroku-rails
# https://rogusdev-docker-heroku-rails.herokuapp.com/

# Look at the "stack" in "Info" on Heroku's settings for this app, under config vars:
#  https://dashboard.heroku.com/apps/rogusdev-docker-heroku-rails/settings

heroku container:push web


vagrant halt
vagrant destroy
```


If you are not in a vagrant vm, you might need to remove postgres:
```
# https://askubuntu.com/questions/32730/how-to-remove-postgres-from-my-installation
dpkg -l | grep postgres

sudo apt-get --purge remove postgresql postgresql-common postgresql-client-common
```

Integrating with CircleCI is possible but actually quite complicated -- these are just stubs I will fill in soon, the links tell the story for now:
```
# https://circleci.com/docs/2.0/deployment_integrations/#heroku
# https://circleci.com/docs/2.0/project-walkthrough/#deploying-to-heroku
cat << EOF1 > .circleci/setup-heroku.sh
#!/bin/bash
wget https://cli-assets.heroku.com/branches/stable/heroku-linux-amd64.tar.gz
sudo mkdir -p /usr/local/lib /usr/local/bin
sudo tar -xvzf heroku-linux-amd64.tar.gz -C /usr/local/lib
sudo ln -s /usr/local/lib/heroku/bin/heroku /usr/local/bin/heroku

cat > ~/.netrc << EOF
machine api.heroku.com
  login $HEROKU_LOGIN
  password $HEROKU_API_KEY
EOF

cat >> ~/.ssh/config << EOF
VerifyHostKeyDNS yes
StrictHostKeyChecking no
EOF
EOF1

# https://circleci.com/docs/2.0/executor-types/#using-docker
cat << EOF > .circleci/config.yml
version: 2
jobs:
  build:
    machine: true
    steps:
      - checkout
      - run:
          name: Greeting
          command: echo Hello, world.
      - run:
          name: Print the Current Time
          command: date
EOF
```
