FROM ruby:3.1.1-bullseye

COPY Gemfile .
COPY Gemfile.lock .

RUN gem install bundler \
 && bundle install