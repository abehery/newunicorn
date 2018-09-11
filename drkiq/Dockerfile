FROM ruby:2.2.3-slim

MAINTAINER Ayman Behery <a.mhbehery@gmail.com>

# Install dependencies:
# - build-essential: To ensure certain gems can be compiled
# - nodejs: Compile assets
# - libpq-dev: Communicate with postgres through the postgres gem
# - postgresql-client-9.4: In case you want to talk directly to postgres
RUN apt-get update && apt-get install -qq -y build-essential nodejs libpq-dev postgresql-client-9.4 --fix-missing --no-install-recommends

# Set an environment variable to store where the app is installed to inside
# of the Docker image.
ENV INSTALL_PATH /drkiq
RUN mkdir -p $INSTALL_PATH

WORKDIR $INSTALL_PATH

COPY Gemfile Gemfile
RUN bundle install

COPY . .

VOLUME ["$INSTALL_PATH/public"]


CMD bundle exec rake RAILS_ENV=production DATABASE_URL=postgresql://user:pass@postgresql/dbname SECRET_TOKEN=pickasecuretoken assets:precompile & bundle exec unicorn -c config/unicorn.rb & bundle exec sidekiq -C config/sidekiq.yml