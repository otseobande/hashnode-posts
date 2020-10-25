## Building a Serverless URL shortener with AWS SAM and DynamoDB

I've been working closely with software infrastructure for a couple of years and I've gotten to enjoy the raw vibe of having to set up a server for a specific purpose or for a particular software.  I like performing experiments on AWS and have learned to love using CloudFormation to configure infrastructure resources on AWS. I enjoy the concept of serverless and I've been taking it out for several spins.

For this article, I'll be exploring AWS Lambda functions using the AWS Serverless Application Model (SAM). When I learned about Lambda functions they blew my mind away and its possible applications were very fascinating but I quickly realized that it was hard to manage and deploy the code for lambda functions for a reasonably sized application because of the process of uploading zip files for application code and managing API Gateway configuration for the functions. Luckily, AWS understood this too and created the SAM service to solve this problem. SAM combines the best of Lambda, API Gateway, and CloudFormation together. 

To explore this technology better, I'll be building a simple URL shortener with it.

### Requirements
- [An AWS account](https://aws.amazon.com/)
- [Node.js v12+](https://nodejs.org/en/download/)
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html)
- [Docker](https://docs.docker.com/get-docker/)
- DynamoDB

### Design

Before we proceed to build this application, let's take a step back and try to visualize how our final result would work. We need a persistent store to record URLs with an id and we'll use DynamoDB to help with that. We need a REST API endpoint for submitting a URL and that endpoint should save it and return the id to access the shortened version with. Finally, we need a function to redirect users to their intended long URLs. This means we would need two endpoints/functions:

- POST /shorten
- GET /{id}

### Setting up the project

To startup, we need to initiate a new project with SAM:

```bash
sam init
```

This should start an interactive session and we'll be using the "AWS Quick Start Templates" with the "nodejs12.xx" runtime and the "Quick Start: From Scratch" template. Our selections should look similar to this:

![Screenshot 2020-10-25 at 15.41.41.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1603637036405/baLI4UHjq.png)

Feel free to name the project whatever you like on the "Project name" prompt.
 
### Create a Dynamo DB Table

To create a Dynamo DB table, login to the AWS console, search for the DynamoDB service, and click "Create Table". Create a table called `short-urls` with a `url` primary key.


![Screenshot 2020-10-25 at 16.05.22.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1603638416012/7QjpHx6FY.png)

Please note that you can also create a DynamoDB table by specifying it as a `Resource` in the `template.yml` file using Cloudformation syntax but for this article, we'll be working with DynamoDB from the AWS console.


### Create the Shorten URL function

Like we discussed above, to shorten the URL we would map that URL with an id and store both in our database.

To create unique short ids we'll be using the `short-unique-id` npm package and we'll be using `aws-sdk` to access DynamoDB. You can install both these packages by running: 

```bash
npm i aws-sdk short-unique-id 
```

In the `src/handlers` directory, let's create a `shorten-url.js` file and export the shortenUrl function. The function would take a URL from the request, create an id for the URL and store both the URL and the id in the database table. Here's how that looks:

```javascript
const { default: ShortUniqueId } = require('short-unique-id');
const AWS = require("aws-sdk");

const uid = new ShortUniqueId();
const docClient = new AWS.DynamoDB.DocumentClient();
const TableName = "short-urls";

exports.shortenUrl = async (event) => {
  const { url } = JSON.parse(event.body);

  const params = {
    TableName,
    Item: {
      url,
      id: uid()
    },
  }

  await docClient.put(params).promise()

  return {
    statusCode: 200,
    body: JSON.stringify({
      data: params.Item
    })
  }
};

```

Now, let's register this function in the `template.yml` to an API path:

```yaml
Resources:
  ShortenUrl:
    Type: AWS::Serverless::Function
    Properties:
      Handler: src/handlers/shorten-url.shortenUrl
      Runtime: nodejs12.x
      Timeout: 100
      Events:
        ShortenUrl:
          Type: Api
          Properties:
            Path: /shorten
            Method: post

```

Okay great, we can now test this by running

```bash
sam local start-api
```

You should be able to make a post request to `localhost:3000/shorten` with a JSON request body like:

```json
{
    "url": "example.com/long/url/path"
}
```

Nice. We are not yet done with this though. Just like with any application that receives input from users we can't trust the input so we need to validate it.

Let's create a function to validate that a valid URL is passed.

```javascript

const validateUrl = (url) => {
  if (url === undefined) {
    return {
      statusCode: 400,
      body: JSON.stringify({
        errors: [
          {
            'field': 'url',
            'message': 'Url is required'
          }
        ]
      })
    }
  }

  try {
    new URL(url);
  } catch (_) {
    return {
      statusCode: 400,
      body: JSON.stringify({
        errors: [
          {
            'field': 'url',
            'message': 'Please provide a valid url'
          }
        ]
      })
    };
  }
}
```

We also need to just return the URL details if it has already been shortened before. We'll use the `getUrlIfExists` function to do that:

```javascript
const getUrlIfExists = async (url) => {
  try {
    const existingUrl = await docClient.get({
      TableName,
      Key: {
        url
      }
    }).promise();

    return Object.keys(existingUrl).length !== 0 || existingUrl;
  } catch (error) {
    return false
  }
}
```
We can now add these functions to the `shortenUrl` handler function

```javascript
exports.shortenUrl = async (event) => {
  const { url } = JSON.parse(event.body);

  const invalidUrlResponse = validateUrl(url);

  if (invalidUrlResponse) {
    return invalidUrlResponse
  }

  const existingUrl = await getUrlIfExists(url);

  if (existingUrl && Object.keys(existingUrl).length !== 0) {
    return {
      statusCode: 200,
      body: JSON.stringify({
        data: existingUrl.Item
      })
    }
  }

  const params = {
    TableName,
    Item: {
      url,
      id: uid(),
      visitCount: 0
    },
  }

  await docClient.put(params).promise()

  return {
    statusCode: 200,
    body: JSON.stringify({
      data: params.Item
    })
  }
}
```

Okay, our `shortenUrl` function is good to go!


### Create the Redirect to Original URL Function

We've been able to register URLs and get IDs mapped to them. Next up we need a function to redirect to the original URLs when a path is visited with that ID. Before we do that, we need to create an index on the ID column on DynamoDB.

![Screenshot 2020-10-25 at 17.37.57.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1603643981754/UKtHi_pAg.png)

Once that's done, we can then create our `redirectUrl` handler function in the `src/handlers` directory.

```javascript
const AWS = require("aws-sdk");

const docClient = new AWS.DynamoDB.DocumentClient();
const TableName = "short-urls";

exports.redirectUrl = async (event) => {
  const { id } = event.pathParameters;

  const queryResponse = await docClient.query({
    TableName,
    IndexName : 'id-index',
    KeyConditionExpression: "id = :v_id",
    ExpressionAttributeValues: {
      ":v_id": id
    },
  }).promise();

  if (queryResponse.Items.length < 1) {
    return jsonResponse({
      status: 404,
      body: {
        message: 'Url not found'
      }
    })
  }

  return {
    statusCode: 301,
    headers: {
      Location: queryResponse.Items[0].url,
    }
  }
};
```

Let's register this handler in the `template.yml` file:

```yaml
RedirectUrl:
    Type: AWS::Serverless::Function
    Properties:
      Handler: src/handlers/redirect-url.redirectUrl
      Runtime: nodejs12.x
      Timeout: 100
      Events:
        ShortenUrl:
          Type: Api
          Properties:
            Path: /{id}
            Method: get
```

And Boom! We have a working URL shortener. Local invocations of SAM might be slow but that isn't the same performance you get once you deploy the functions.

### Conclusion
I had fun doing this and I got to understand how to build applications using AWS SAM. I definitely see myself using it to solve problems in the future.
 
### References
- [DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html)
- [AWS SAM Developer Guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html)