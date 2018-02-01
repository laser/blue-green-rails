FROM ruby:2.5.0-alpine

RUN apk add --update \
  alpine-sdk \
  nodejs \
  postgresql-dev \
  tzdata

COPY Gemfile* /opt/bundle/
WORKDIR /opt/bundle
RUN bundle install

COPY . /opt/app
WORKDIR /opt/app

ENTRYPOINT ["/bin/ash", "-c"]