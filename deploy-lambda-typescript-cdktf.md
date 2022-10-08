# Goal

Goal behind this blog, is to deploy a `lambda` function in `aws` written in `typescript` using [cdktf](https://www.terraform.io/cdktf) and [esbuild](https://esbuild.github.io/) and make that `lambda` function invocable from internet. 

# Prerequisite

I assume you have

- Basic `typescript` knowledge
- `git`, `node`, `npm` and `terraform cli` installed
- `aws` account and cli installed and configured with `access_key_id` and `access_key_secret`

# Setup

Clone the repo `nivekithan/blog-deploy-typescript-lambda`

```bash
git clone https://github.com/nivekithan/blog-deploy-typescript-lambda.git lambda-cdktf
cd lambda-cdktf
```

# Familiarizing with the `ts-lambda` source code

If you see the code you will find two directories named `cdktf` and `ts-lambda`. `ts-lambda` contains source code for our `lambda` function and `cdktf` contains source for managing our infrastructure. 

Our `lambda` function in `ts-lambda/src/index.ts` is fairly simple 


```ts
export const handler = async () => {
  return {
    statusCode: 200,
    headers: {
      "Content-Type": "text/html; charset=utf-8",
    },
    body: `<h1>Hello World, This is written in typescript</h1>`,
  };
};
```

It exports a `handler` function which when called will return following `html`

```html
<h1>Hello World, This is written in typescript</h1>
```

# Deploying our application

We are not actually intreseted in what does our `lambda` function does. We are only intreseted in deploying that `lambda` function. So lets look into `cdktf` directory. 

First lets see contents of `package.json`


```ts
// cdktf/package.json

{
  "name": "cdktf",
  "version": "1.0.0",
  "description": "",
  "keywords": [],
  "author": "",
  "license": "MIT",
  "dependencies": {
    "@cdktf/provider-aws": "^9.0.51",
    "cdktf": "^0.12.3",
    "constructs": "^10.1.123",
    "esbuild": "^0.15.10"
  },
  "devDependencies": {
    "@types/node": "^18.8.0"
  }
}
```

- `cdktf` package allows us to manage our infrastructure in `typescript` instead of `HCL` 
- `@cdktf/provider-aws` package is used to provision and manage `aws` infrastructure
- `constructs` package is used to create reusable cloud resource
- `esbuild` package is used to compile `typescript` to `javascript` and bundle it to single file.


Lets take a look at `main.ts`

```ts
import { Construct } from "constructs";
import { App, TerraformOutput, TerraformStack } from "cdktf";
import * as aws from "@cdktf/provider-aws";
import path from "node:path";
import { TypescriptFunction } from "./lib/typescriptFunction";

const lambdaRolePolicy = {
  Version: "2012-10-17",
  Statement: [
    {
      Action: "sts:AssumeRole",
      Principal: {
        Service: "lambda.amazonaws.com",
      },
      Effect: "Allow",
      Sid: "",
    },
  ],
};

export type LambdaConfig = {
  /**
   * Absolute path to directory which contains src/index.js which then exports handler
   * function
   */
  path: string;

  /**
   * Lambda version
   */
  version: string;
};

class LambdaStack extends TerraformStack {
  constructor(scope: Construct, name: string, config: LambdaConfig) {
    super(scope, name);

    new aws.AwsProvider(this, "aws", {
      region: "ap-south-1",
    });

    const sourceCodeAsset = new TypescriptFunction(this, "lambda-source-code", {
      absPath: config.path,
      handler: "index.handler",
    });

    // Create a s3 bucket
    const bucket = new aws.s3.S3Bucket(this, "bucket", {
      bucketPrefix: "slack-search-lambda",
    });

    // Upload source code to s3
    const lambdaArchive = new aws.s3.S3Object(this, "lambda-archive", {
      bucket: bucket.bucket,
      key: `${sourceCodeAsset.asset.fileName}/${config.version}`,
      source: sourceCodeAsset.asset.path,
    });

    // Create lambda role

    const role = new aws.iam.IamRole(this, "lambda-exec", {
      name: `learn-cdktf-${name}`,
      assumeRolePolicy: JSON.stringify(lambdaRolePolicy),
    });

    new aws.iam.IamRolePolicyAttachment(this, "lambda-managed-policy", {
      policyArn:
        "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole",
      role: role.name,
    });

    const lambdaFunc = new aws.lambdafunction.LambdaFunction(
      this,
      "slack-search-lambda",
      {
        functionName: `slack-search-lambda`,
        s3Bucket: bucket.bucket,
        s3Key: lambdaArchive.key,
        handler: "index.handler",
        runtime: "nodejs16.x",
        role: role.arn,
      }
    );

    new aws.lambdafunction.LambdaPermission(
      this,
      "lambda-allow-public-access",
      {
        statementId: "FunctionURLAllowPublicAccess",
        principal: "*",
        action: "lambda:InvokeFunctionUrl",
        functionName: lambdaFunc.functionName,
        functionUrlAuthType: "NONE",
      }
    );

    const lambdaFunctionUrl = new aws.lambdafunction.LambdaFunctionUrl(
      this,
      "lambdaFunctionUrl",
      { authorizationType: "NONE", functionName: lambdaFunc.functionName }
    );

    new TerraformOutput(this, "lambda-url", {
      value: lambdaFunctionUrl.functionUrl,
    });
  }
}

const app = new App();

new LambdaStack(app, "cdktf", {
  path: path.resolve(__dirname, "..", "ts-lambda"),
  version: "0.0.5",
});

app.synth();
```

# Deploying to `aws`

To deploy to `aws` first we have to install `cdktf` cli. For that run 


```bash
pnpm i -g cdktf-cli

# if you prefer npm
npm i -g cdktf-cli
```

then install all packages in `cdktf` directory

```bash
# lambda-cdktf/cdktf/

pnpm i

# or if you prefer npm
npm i
```

Now to deploy run

```bash
# lambda-cdktf/cdktf
cdktf deploy
```

Then select `Approve` and thats it. Once the its deployed you will be able to see the `lambda` url.

For example, for me it shows

```bash
Apply complete! Resources: 7 added, 0 changed, 0 destroyed.
       
Outputs:

    lambda-url = "https://3gcd3p4j4ypvucb5eo4bg3dcgm0dobsb.lambda-url.ap-south-1.on.aws/"

cdktf
lambda-url = https://3gcd3p4j4ypvucb5eo4bg3dcgm0dobsb.lambda-url.ap-south-1.on.aws/
```

Opening that url in browser shows

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1665250262657/j5SkZddOQ.png align="left")

Hopefully it shows same for you too. If you face any let me know, I will help you to my best of efforts.    

# Cleanup

To destroy all the resource created and prevent accidental aws cost run 

```bash
cdktf destroy
```