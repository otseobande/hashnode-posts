## Setting up Cypress for integration tests for a NextJS app on Gitlab CI

## Introduction

Before we begin, I think it's necessary to highlight the importance of integration tests and why we decided to use Cypress. I and my team noticed that we kept breaking stuff as we added more and more features to the application. To solve that, we decided to write more unit tests. After a few weeks, we noticed that unit tests alone were not really catching bugs that related to the application feature flows and this is why we decided to add integration tests to the mix. Doing this has boosted the confidence we have in our tests and the application in general. We chose Cypress because of the ease of setup and the solid documentation it had. We found it easy to use too.

## Setting up dependencies

The application is a NextJS application powered with an AdonisJS API application. The first problem I had to solve was how to get both of these application servers running while also running the Cypress test in the same Gitlab CI job. I also had to make provision for the API application dependency services which were Postgres and Redis in this case. After hours of brainstorming, I got started by configuring the services the API needed using [Gitlab CI services](https://docs.gitlab.com/ee/ci/services/). Adding the job to the `.gitlab-ci.yml` looked like:

```yaml
integration_test:
  image: node:10.12.0
  stage: test
  variables:
    POSTGRES_DB: test_database
    POSTGRES_USER: runner
    POSTGRES_PASSWORD: ""
  services:
    - postgres:11.1
    - redis:latest
```

Cypress requires certain packages to be installed on a Linux CI container to be able to run. They included them in their [Continous integration documentation](https://docs.cypress.io/guides/guides/continuous-integration.html#Advanced-setup). We also need to install Adonis CLI globally to run adonis commands for the API. A `before_script` would be used to set these up.

```yaml
# ...
  before_script:
      - apt-get update
      - apt-get install -y libgtk2.0-0 libgtk-3-0 libnotify-dev libgconf-2-4 libnss3 libxss1 libasound2 libxtst6 xauth xvfb zip unzip
      - npm i -g @adonisjs/cli@4.0.10 cypress
# ...
```

## Starting the NextJS server

Next, we install the NextJS npm dependencies and start the server in the background. I decided to also do this in the `before_script` because I also consider it part of the job setup.

```yaml
# ... before_script
    - npm install
    - npm run build
    - ( npm run start </dev/null &>/dev/null & ) 
```

The last line immediately stands out because it looks strange at first. In summary, it starts the server in a subshell and detaches it's `stdout` and `stderr` from the current shell. You can read more about it in this [StackOverflow Answer](https://unix.stackexchange.com/questions/497207/difference-between-dev-null-21-and-dev-null-dev-null). 

## Starting the API application

After getting our NextJS server running we need to start the API application. It won't really be an integration test if we don't test how it behaves with our API application app connected to it. So we would be starting it also. The first thing is to clone the API application to get the latest code. This is also done in the `before_script`. We'll also be installing it's npm dependencies, running database migration and seeds, and finally starting the API server.

```yaml
# ... before_script
    - git clone https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.com/org-name/api-app
    - cd api-app
    - npm i
    - npm run migrate
    - npm run seed
    - ( npm run start </dev/null &>/dev/null & )
    - cd ../
```

## Putting it all together and running the tests

Additionally, because Cypress tests can get flaky we don't want a failure of the integration_test to break the pipeline and prevent review apps from deploying or any important job in the next stage so we added `allow_failure: true` to the job. Feel free to remove this if it's not necessary for your pipeline. Also, failed Cypress tests generate screenshots and this is useful when debugging. To get that from the CI pipeline, screenshots are uploaded as Gitlab-CI artifacts. The full job looks something like this:

```yaml
integration_test:
  image: node:10.12.0
  stage: test
  variables:
    POSTGRES_DB: test_database
    POSTGRES_USER: runner
    POSTGRES_PASSWORD: ""
  services:
    - postgres:11.1
    - redis:latest
  before_script:
      - apt-get update
      - apt-get install -y libgtk2.0-0 libgtk-3-0 libnotify-dev libgconf-2-4 libnss3 libxss1 libasound2 libxtst6 xauth xvfb zip unzip
      - npm i -g @adonisjs/cli@4.0.10
      - npm install
      - npm run build
      - ( npm run start </dev/null &>/dev/null & )
      - git clone https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.com/org-name/api-app
      - cd api-app
      - npm I
      - npm run migrate
      - npm run seed
      - ( npm run start </dev/null &>/dev/null & )
      - cd ../
   script:
      - cypress run
   artifacts:
      paths:
         - cypress/screenshots/
      when: always
   expire_in: 1 week
   allow_failure: true
   only:
     - master
     - merge_requests
```

## Conclusion

So there you have it, that's how I got Cypress setup on Gitlab-CI. I know there are other techniques that can be used to get this setup done. I'm curious to know how you set up yours. Feel free to leave a comment.