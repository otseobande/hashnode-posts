## Deploying Heroku Review Apps on Gitlab.

A few weeks ago, I was working with a team and we noticed that the code review stage was getting slower as more merge requests was created by developers. One bottleneck in the review stage was that members of the team had to run each MR branch locally to test functionality. I decided to take on the task to help make our lives easier by automating the deployment of a review app for every MR opened for the project. The project repository is hosted on Gitlab so by default we used Gitlab-CI as our pipeline tool. To simplify the deployment, I chose **Heroku** because Heroku dyno deployment is simple and using free dynos would be a lot cheaper than other alternatives.

Before this, I've had experience setting up Heroku review apps on Github and the ease of setup was almost plug and play but Heroku doesn't have support for review apps integration with Gitlab for now so it had to be a manual process sadly.

The first step I took was to plan the format for naming each review app. I decided to use `[projectname]-mr-[MR_ID]`. Since MRs are unique, this was a perfect app name format for Heroku. Based on the chosen naming format, I needed the current Merge Request Id to uniquely name each review app. Gitlab-CI comes with  [predefined environment variables](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html) that come to the rescue in situations like this. After a quick scan I found `CI_MERGE_REQUEST_IID` which was perfect for this use case.

This app uses NextJs and the `.gitlab-ci.yml` file configuration used to run tests looks like this. For the purpose of this article the app would be referred to as `testapp`.

```
image: node:10.12.0

stages:
  - test

test:
  stage: test
  before_script:
    - npm i -g codecov
    - npm install
  script:
    - npm test
    - codecov
``` 

To update the CI pipeline to run review app deployment, two extra CI stages are needed. One to start the deployment and another to delete the deployed app once the merge request is closed. Let's quickly go through the stage needed to deploy the review app.

### App deployment

I named the stage to initiate the review app deployment `start_review` and it uses Heroku API to initiate an app creation. To authenticate API requests to Heroku, a key should be generated from the Heroku dashboard and saved in the projects CI environment variables. I saved mine as `HEROKU_API_KEY`. Next, the API KEY is added to the `Authorization` header using *curl* as the request client. The app name is also constructed using the environment variable and sent in the request body along with the deployment region.

```
# ...other CI stages
start_review:
  stage: review
  script:
    - >-
      curl -X POST https://api.heroku.com/apps
      -H "Accept: application/vnd.heroku+json; version=3"
      -H "Authorization: Bearer $HEROKU_API_KEY"
      -H "Content-Type: application/json"
      -d "{
        \"name\":\"testapp-mr-${CI_MERGE_REQUEST_IID}\",
        \"region\": \"eu\"
      }"
```

This creates an empty Heroku app that still needs the current app files to be deployed to it. We'll be using Heroku's git deployment strategy for this. One more thing we need to take note of is that our review app would be needing some environment variables. Using Heroku's review app on Github, this can be easily configured using an `app.json` file but since this isn't the case for Gitlab, I used a hacky method of adding an `.env` file to the app and committing it to the Heroku repo. I needed to remove the line that excludes `.env` from the `.gitignore` file to make this happen. Necessary environment variables were also saved to the CI environment so it was easy to use `printenv` to write to a `.env` file.  Once this is done the `start_review` stage commits the file using the CI's git credentials and pushes to the Heroku repository to initiate a deployment. These scripts were used to achieve the needed objectives:

```
# ...
    - git checkout -B $CI_COMMIT_REF_SLUG
    - git config user.email $GITLAB_USER_EMAIL
    - git config user.name $GITLAB_USER_NAME
    - touch .env
    - printenv >> .env
    - sed -i '/.env/d' .gitignore
    - git add .
    - git commit -m "add .env file"
    - git push -f https://heroku:$HEROKU_API_KEY@git.heroku.com/testapp-mr-${CI_MERGE_REQUEST_IID}.git $CI_COMMIT_REF_SLUG:master
```

The `script` section of the stage is complete and is almost ready to deploy our review app now. We still need to add some properties to the `start_review` stage to configure it properly. One of which is the `environment` property. The environment property specifies the name of the environment, the url to access the application deployed to that environment  and the Gitlab CI stage to run when the environment is torn down. For the environment name I used the format `review/[branch-name]`. To get the branch name, I used the Gitlab CI predefined variable `CI_BUILD_REF_NAME`. The format for the url has been discussed already. The code for looks like this:

```
# ...other start_review properites
  environment:
    name: review/$CI_BUILD_REF_NAME
    url: https://testapp-mr-${CI_MERGE_REQUEST_IID}.herokuapp.com
    on_stop: stop_review
```

The final properties to add to the `start_review` stage are conditional properites `only` and `except`. This enables the stage to run only on `merge_requests` and not run on push to `master`. So we add the following to the `start_review` stage:

```
# ... other start_review properties.
only:
   - merge_requests
except:
   - master
```

The complete `start_review` stage looks like this:

```
start_review:
  stage: review
  script:
    - >-
      curl -X POST https://api.heroku.com/apps
      -H "Accept: application/vnd.heroku+json; version=3"
      -H "Authorization: Bearer $HEROKU_API_KEY"
      -H "Content-Type: application/json"
      -d "{
        \"name\":\"testapp-mr-${CI_MERGE_REQUEST_IID}\",
        \"region\": \"eu\"
      }"
    - git checkout -B $CI_COMMIT_REF_SLUG
    - git config user.email $GITLAB_USER_EMAIL
    - git config user.name $GITLAB_USER_NAME
    - touch .env
    - printenv >> .env
    - sed -i '/.env/d' .gitignore
    - git add .
    - git commit -m "add .env file"
    - git push -f https://heroku:$HEROKU_API_KEY@git.heroku.com/testapp-${CI_MERGE_REQUEST_IID}.git $CI_COMMIT_REF_SLUG:master
  environment:
    name: review/$CI_BUILD_REF_NAME
    url: https://testapp-${CI_MERGE_REQUEST_IID}.herokuapp.com
    on_stop: stop_review
  only:
    - merge_requests
  except:
    - master
```

### App tear down

The final piece of the puzzle is the `stop_review` stage that's specified in the `on_stop` property of the `start_review` stage environment. In this stage the pipeline calls the Heroku API to delete the Heroku app.

```
stop_review:
  stage: review
  script:
    - >-
      curl
      -X DELETE
      https://api.heroku.com/apps/testapp-mr-${CI_MERGE_REQUEST_IID}
      -H "Content-Type: application/json"
      -H "Accept: application/vnd.heroku+json; version=3"
      -H "Authorization: Bearer $HEROKU_API_KEY"
  when: manual
  environment:
    name: review/$CI_BUILD_REF_NAME
    action: stop
  only:
    - merge_requests
  except:
    - master
```

### Complete config file


The complete `.gitlab-ci.yml` with all the stages looks like this:

```
image: node:10.12.0

stages:
  - test
  - review
  - deploy

test:
  stage: test
  before_script:
    - npm i -g codecov
    - npm install
  script:
    - npm test
    - codecov
  only:
    - master
    - merge_requests

start_review:
  stage: review
  script:
    - >-
      curl -X POST https://api.heroku.com/apps
      -H "Accept: application/vnd.heroku+json; version=3"
      -H "Authorization: Bearer $HEROKU_API_KEY"
      -H "Content-Type: application/json"
      -d "{
        \"name\":\"testapp-mr-${CI_MERGE_REQUEST_IID}\",
        \"region\": \"eu\"
      }"
    - git checkout -B $CI_COMMIT_REF_SLUG
    - git config user.email $GITLAB_USER_EMAIL
    - git config user.name $GITLAB_USER_NAME
    - touch .env
    - printenv >> .env
    - sed -i '/.env/d' .gitignore
    - git add .
    - git commit -m "add .env file"
    - git push -f https://heroku:$HEROKU_API_KEY@git.heroku.com/testapp-mr-${CI_MERGE_REQUEST_IID}.git $CI_COMMIT_REF_SLUG:master
  environment:
    name: review/$CI_BUILD_REF_NAME
    url: https://testapp-mr-${CI_MERGE_REQUEST_IID}.herokuapp.com
    on_stop: stop_review
  only:
    - merge_requests
  except:
    - master

stop_review:
  stage: review
  script:
    - >-
      curl
      -X DELETE
      https://api.heroku.com/apps/testapp-mr-${CI_MERGE_REQUEST_IID}
      -H "Content-Type: application/json"
      -H "Accept: application/vnd.heroku+json; version=3"
      -H "Authorization: Bearer $HEROKU_API_KEY"
  when: manual
  environment:
    name: review/$CI_BUILD_REF_NAME
    action: stop
  only:
    - merge_requests
  except:
    - master
```

##### References
- https://www.martinlugton.com/how-to-create-review-apps-in-heroku-from-gitlab/