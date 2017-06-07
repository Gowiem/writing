# Hippocrates Deployment/CI Info

## Goals
1. Flush out the deployment strategy for Rails, Ember CLI, Postgres to AWS
2. Flush out ability to run tests in CI environment

## Ember Specifics

[ember-cli-deploy](https://github.com/ember-cli-deploy/ember-cli-deploy) is the go to lib for Ember Deployment. It is a framework for applying plugins to a deployment pipeline. This combined with [ember-cli-deploy-lightning-pack](https://github.com/ember-cli-deploy/ember-cli-deploy-lightning-pack) lays out a complex, but fairly out of the box solution for deploying the Ember side of the application. Assets live in S3 (fronted by cloudfront), the index.html file is pushed to Redis, Rails serves the index.html file out of Redis and hosts the API.

IMG

## Rails Specifics

Following [this blog post](https://nickjanetakis.com/blog/dockerize-a-rails-5-postgres-redis-sidekiq-action-cable-app-with-docker-compose) we can pull the relevant bits out and use them. Specifically, we'll just be deploying a Rails, Redis, and Postgres backed application with Docker. The Postgres and Redis sides of things will be thrown away for Production as we should be using AWS RDS + ElastiCache.

Use AWS's Elastic Beanstalk to deploy a [Single Container Docker](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/docker-singlecontainer-deploy.html).

## Required to implement

1. Access to ElasticBeanstalk, ElastiCache, EC2, RDS, S3
2. Access to a bastion box which we can open the security group so it can be used to SSH Tunnel the index.html file upload

## Testing

1. Rails app will be testable through:
```
docker-compose up
docker-compose run -e "RAILS_ENV=test" app rake db:create db:migrate
docker-compose run -e "RAILS_ENV=test" app rake test
```

2. `ember test` can be setup via the [`danlynn/ember-cli`](https://hub.docker.com/r/danlynn/ember-cli/) Docker Container -- https://ember-cli.com/user-guide/#testing
