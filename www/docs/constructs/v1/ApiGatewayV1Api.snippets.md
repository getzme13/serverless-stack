### Configuring routes

#### Adding catch-all route

Add routes after the API has been created.

```js {4}
const api = new ApiGatewayV1Api(this, "Api", {
  routes: {
    "GET /notes"   : "src/list.main",
    "ANY /{proxy+}": "src/catch.main",
  },
});
```

#### Lazily adding routes

Add routes after the API has been created.

```js
const api = new ApiGatewayV1Api(this, "Api", {
  routes: {
    "GET    /notes": "src/list.main",
    "POST   /notes": "src/create.main",
  },
});

api.addRoutes(this, {
  "GET    /notes/{id}": "src/get.main",
  "PUT    /notes/{id}": "src/update.main",
  "DELETE /notes/{id}": "src/delete.main",
});
```

#### Specifying function props for all the routes

You can extend the minimal config, to set some function props and have them apply to all the routes.

```js {2-8}
new ApiGatewayV1Api(this, "Api", {
  defaults: {
    function: {
      timeout: 20,
      environment: { tableName: table.tableName },
      permissions: [table],
    },
  },
  routes: {
    "GET  /notes": "src/list.main",
    "POST /notes": "src/create.main",
  },
});
```

#### Configuring an individual route

Configure each Lambda function separately.

```js
new ApiGatewayV1Api(this, "Api", {
  routes: {
    "GET /notes": {
      function: {
        handler: "src/list.main",
        timeout: 20,
        environment: { tableName: "NOTES_TABLE" },
        permissions: [table],
      },
    },
  },
});
```

Note that, you can set the `defaults.function` while using the `function` per route. The `function` will just override the `defaults.function`. Except for the `environment`, the `layers`, and the `permissions` properties, which will be merged.

```js
new ApiGatewayV1Api(this, "Api", {
  defaults: {
    function: {
      timeout: 20,
      environment: { tableName: "NOTES_TABLE" },
    },
  },
  routes: {
    "GET /notes": {
      function: {
        handler: "list.main",
        timeout: 10,
        environment: { bucketName: "NOTES_BUCKET" },
      },
    },
    "POST /notes": "create.main",
  },
});
```

So in the above example, the `GET /notes` function doesn't use the `timeout` that is set in the `defaults.function`. It'll instead use the one that is defined in the function definition (`10 seconds`). And the function will have both the `tableName` and the `bucketName` environment variables set.

#### Attaching permissions for the entire API

Allow the entire API to access S3.

```js {11}
const api = new ApiGatewayV1Api(this, "Api", {
  routes: {
    "GET    /notes"     : "src/list.main",
    "POST   /notes"     : "src/create.main",
    "GET    /notes/{id}": "src/get.main",
    "PUT    /notes/{id}": "src/update.main",
    "DELETE /notes/{id}": "src/delete.main",
  },
});

api.attachPermissions(["s3"]);
```

#### Attaching permissions for a specific route

Allow one of the routes to access S3.

```js {11}
const api = new ApiGatewayV1Api(this, "Api", {
  routes: {
    "GET    /notes"     : "src/list.main",
    "POST   /notes"     : "src/create.main",
    "GET    /notes/{id}": "src/get.main",
    "PUT    /notes/{id}": "src/update.main",
    "DELETE /notes/{id}": "src/delete.main",
  },
});

api.attachPermissionsToRoute("GET /notes", ["s3"]);
```

#### Getting the function for a route

```js {11}
const api = new ApiGatewayV1Api(this, "Api", {
  routes: {
    "GET    /notes"     : "src/list.main",
    "POST   /notes"     : "src/create.main",
    "GET    /notes/{id}": "src/get.main",
    "PUT    /notes/{id}": "src/update.main",
    "DELETE /notes/{id}": "src/delete.main",
  },
});

const listFunction = api.getFunction("GET /notes");
```

### Custom domains

You can also configure the API with a custom domain. SST currently supports domains that are configured using [Route 53](https://aws.amazon.com/route53/). If your domains are hosted elsewhere, you can [follow this guide to migrate them to Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/MigratingDNS.html).

#### Using the basic config

```js {2}
new ApiGatewayV1Api(this, "Api", {
  customDomain: "api.domain.com",
  routes: {
    "GET /notes": "src/list.main",
  },
});
```

#### Using the full config

```js {2-7}
new ApiGatewayV1Api(this, "Api", {
  customDomain: {
    domainName: "api.domain.com",
    hostedZone: "domain.com",
    endpointType: "edge",
    path: "v1",
  },
  routes: {
    "GET /notes": "src/list.main",
  },
});
```

#### Mapping multiple APIs to the same domain

```js {11-13}
const usersApi = new ApiGatewayV1Api(this, "UsersApi", {
  customDomain: {
    domainName: "api.domain.com",
    path: "users",
  },
});

new ApiGatewayV1Api(this, "PostsApi", {
  customDomain: {
    path: "posts",
    cdk: {
      domainName: usersApi.cdk.domainName,
    },
  },
});
```

#### Importing an existing API Gateway custom domain

```js {6-12}
import { DomainName } from "aws-cdk-lib/aws-apigateway";

new ApiGatewayV1Api(this, "Api", {
  customDomain: {
    path: "newPath",
    cdk: {
      domainName: DomainName.fromDomainNameAttributes(this, "MyDomain", {
        domainName,
        domainNameAliasHostedZoneId,
        domainNameAliasTarget,
      }),
    },
  },
  routes: {
    "GET /notes": "src/list.main",
  },
});
```

#### Importing an existing certificate

```js {6-8}
import { Certificate } from "aws-cdk-lib/aws-certificatemanager";

new ApiGatewayV1Api(this, "Api", {
  customDomain: {
    domainName: "api.domain.com",
    cdk: {
      certificate: Certificate.fromCertificateArn(this, "MyCert", certArn),
    },
  },
  routes: {
    "GET /notes": "src/list.main",
  },
});
```

#### Loading domain name from SSM parameter

If you have the domain name stored in AWS SSM Parameter Store, you can reference the value as the domain name:

```js {3,6-9}
import { StringParameter } from "aws-cdk-lib/aws-ssm";

const rootDomain = StringParameter.valueForStringParameter(this, `/myApp/domain`);

new ApiGatewayV1Api(this, "Api", {
  customDomain: {
    domainName: `api.${rootDomain}`,
    hostedZone: rootDomain,
  },
  routes: {
    "GET /notes": "src/list.main",
  },
});
```

Note that, normally SST will look for a hosted zone by stripping out the first part of the `domainName`. But this is not possible when the `domainName` is a reference. Since its value will be resolved at deploy time. So you'll need to specify the `hostedZone` explicitly.

### Authorization

You can use IAM or JWT to add auth to your APIs.

#### Adding IAM authorization

You can secure your APIs (and other AWS resources) by setting the `defaults.authorizer`.

```js {2-4}
new ApiGatewayV1Api(this, "Api", {
  defaults: {
    authorizer: "iam",
  },
  routes: {
    "GET  /notes": "list.main",
    "POST /notes": "create.main",
  },
});
```

#### Adding IAM authorization to a specific route

You can also secure specific routes in your API.

```js {5}
new ApiGatewayV1Api(this, "Api", {
  routes: {
    "GET /public": "src/public.main",
    "GET /private": {
      authorizer: "iam",
      function: "src/private.main",
    },
  },
});
```

#### Adding Lambda authorization

You can also use a Lambda function to authorize users to access your API. Note that, this is a different authorization method when compared to using IAM, which allows you to secure other AWS resources as well.

```js
import * as apigateway from "aws-cdk-lib/aws-apigateway";

new ApiGatewayV1Api(this, "Api", {
  authorizers: {
    myAuthorizer: {
      type: "lambda_request",
      function: new Function(this, "Authorizer", {
        handler: "src/authorizer.main",
      }),
      identitySources: [apigateway.IdentitySource.header("Authorization")],
    },
  },
  defaults: {
    authorizer: "myAuthorizer",
  },
  routes: {
    "GET  /notes": "src/list.main",
    "POST /notes": "src/create.main",
  },
});
```

#### Adding Lambda authorization to a specific route

You can also secure specific routes by setting the `authorizer` per route.

```js {16}
import * as apigateway from "aws-cdk-lib/aws-apigateway";

new ApiGatewayV1Api(this, "Api", {
  authorizers: {
    myAuthorizer: {
      type: "lambda_request",
      function: new Function(this, "Authorizer", {
        handler: "src/authorizer.main",
      }),
      identitySources: [apigateway.IdentitySource.header("Authorization")],
    },
  },
  routes: {
    "GET /public": "src/public.main",
    "GET /private": {
      authorizer: "myAuthorizer",
      function: "src/private.main",
    },
  },
});
```

#### Using Cognito User Pool as the authorizer

You can also use Cognito User Pools as an authorizer.

```js
new ApiGatewayV1Api(this, "Api", {
  authorizers: {
    myAuthorizer: {
      type: "user_pools",
      userPoolIds: [userPool.userPoolId],
    }
  },
  defaults: {
    authorizer: "myAuthorizer",
    authorizationScopes: ["user.id", "user.email"],
  },
  routes: {
    "GET  /notes": "src/list.main",
    "POST /notes": "src/create.main",
  },
});
```

### Access log

#### Configuring the access log format

Use a CSV format instead of default JSON format.

```js {2-3}
new ApiGatewayV1Api(this, "Api", {
  accessLog:
    "$context.identity.sourceIp,$context.requestTime,$context.httpMethod,$context.routeKey,$context.protocol,$context.status,$context.responseLength,$context.requestId",
  routes: {
    "GET /notes": "src/list.main",
  },
});
```

#### Configuring the log retention setting

```js {2-4}
new ApiGatewayV1Api(this, "Api", {
  accessLog: {
    retention: "one_week",
  },
  routes: {
    "GET /notes": "src/list.main",
  },
});
```

### CORS

Override the default behavior of allowing all methods, and only allow the GET method.

```js {3-5}
new ApiGatewayV1Api(this, "Api", {
  cdk: {
    restApi: {
      defaultCorsPreflightOptions: {
        allowOrigins: ['"*"'],
      },
    },
  },
  routes: {
    "GET /notes": "src/list.main",
  },
});
```

### Advanced examples

#### Configuring Regional endpoint

Configure the internally created CDK `RestApi` instance.

```js {4-10}
import { EndpointType } from "aws-cdk-lib/aws-apigateway";

new ApiGatewayV1Api(this, "Api", {
  cdk: {
    restApi: {
      endpointConfiguration: {
        types: [EndpointType.REGIONAL],
      },
    },
  },
  routes: {
    "GET /notes": "src/list.main",
  },
});
```

#### Importing an existing Rest Api

Override the internally created CDK `RestApi` instance.

```js {4-13}
import { RestApi } from "aws-cdk-lib/aws-apigateway";

new ApiGatewayV1Api(this, "Api", {
  cdk: {
    restApi: RestApi.fromRestApiAttributes(this, "MyRestApi", {
      restApiId,
      rootResourceId,
    }),
    importedPaths: {
      "/notes": "slx2bn",
      "/users": "uu8xs3",
    },
  },
  routes: {
    "GET /notes/{noteId}": "src/getNote.main",
    "GET /users/{userId}": "src/getUser.main",
  },
});
```
