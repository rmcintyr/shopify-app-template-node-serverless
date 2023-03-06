# shopify-app-template-node-serverless
This fork of the Shopify Node App template may be used as a base for building applications that will run on AWS Lambda, using the Serverless Framework

No changes are required to the standard frontend module, but you will need to either clone it, or import your own frontend app, and ensure it builds to frontend/dist.  The app does not perform server-side rendering in Lambda

Changes to the original repo include:
- updates to `package.json` and `web/package.json` to reference new dependencies in the base project and the backend web module
- additon of template serverless framework files to the "backend" web module
  - `serverless.yml` - main serverless framework application definition file 
  - `Dockerfile` - added to build a docker image based on the standard AWS Node runtime image.  The dockerfile included in the project base was not useable for this, and is still useful for deploying to fly.io
  - `.gitignore` - added serverless framework build output/state tracking files
  - `.dockerignore` - prevents unnecessary files being pulled into the docker image
- `web/index.js` has been updated to conditionally use serverless-http for rendering instead of starting an express server

There are some issues outlined below, particularly with regards to session handling that will need to be resolved in your project prior to production use, but the basics are in place to gain familiarity with the deployment model and tools, and to start building a real app from.

## Pre-requisites

### Shopify Partner Account
Please follow the standard instructions in README.md to ensure you are logged in to your Shopify Partner account.  Simply running `shopify app dev` will get you started.

### AWS & Serverless.com Accounts

You will need to ensure you have AWS & Serverless accounts set up and functioning prior to attempting to deploy.  No account details are required in `.env` or elsewhere within the project - you will need to run `serverless login` and `serverless --org=<your-org-name>`

### Domains

Need to ensure you have a public Route53 Hosted Zone, and a wildcard certificate created under ACM in the AWS region you'll be deploying to.  The app will be deployed to stage.domain, e.g. dev.yourcoolapp.io.

You will need to define the domain name either in web/.env, or as a parameter through the serverless dashboard.

### Secure Parameters...

Either use `.env` (or `.env.stage`) files in the web directory (.gitignore is already set to not pass these through), or store these securely in the serverless dashboard, [following instructions here](https://www.serverless.com/framework/docs/guides/parameters)
Remember that you'll probably need seperate `SHOPIFY_API_KEY`/`SHOPIFY_API_SECRET` keys for production and non-production apps so don't try to define these globally!

### Optionals

ngrok setup is optional but highly recommended - dynamic URLs are fine for default shopify dev & debug use but when you move on to testing with serverless-offline, its certainly more convenient to have a static domain name.
Add your ngrok authentication token to `.env` or as a serverless dashboard param, and uncomment/update the appropriate lines in `serverless.yaml`.
You can add new scripts to your top level package.json to simplify setting the application's URL and redirect URLs.

## Setup

Once the prerequisites listed above are in place, you need to make the following changes:

- package.json
    Update the `name`, `version` and `license` parameters to suit your requirements.  You may also update the script for updateUrls to contain your app's domain name.  Domains will be dynamically constructed using the serverless stage, serverless service name and your domain, e.g. `https://dev-shopify-app-template.yourdomain.io`  Duplicate this line if necessary to cover production or other stages you intend to use.
- web/serverless.yml
    Update the `app` and `service` lines at the top of the file.  After running `serverless --org=<your org name>`
- web/.env
  Ensure this file exists (e.g. `touch web/.env`) and contains the following

  ```shell
    SHOPIFY_API_KEY=<your app's shopify api key>
    SHOPIFY_API_SECRET=<your app's shopify api secret>
    ngrok_authtoken=<your personal ngrok authentication token>
    PORT=3000
    FRONTEND_PORT=3000
    BACKEND_PORT=3000
    domain=yourdomain.io
    scopes=<any shopify auth scopes your app requires, e.g write_products>
  ```

## Testing with Serverless Offline
See Known Issues below - the serverless.yaml file must be updated not use docker before proceeding.
From the project root directory, first run `npm run build`, then change to the web directory and run `npm run serverless:offline`

```
npm run dev
cd web
npm run serverless:offline
```

You will need to update your Application URLs in the Shopify Partner Dashboard.  Alternatively, you may add a custom script to the root project's 'package.json' file's scripts block, e.g.

```json
scripts: {
    ...
    "updateUrls:offline": "shopify app update-url --app-url https://shopify-app-template.eu.ngrok.io --redirect-urls https://shopify-app-template.eu.ngrok.io/auth/callback,https://shopify-app-template.eu.ngrok.io/auth/shopify/callback,https://shopify-app-template.eu.ngrok.io/api/auth/callback"
    ...
}
```

## Deploying to AWS via Serverless
From the project root directory, first run `npm run build`, then change to the web directory and run `serverless deploy`

```
npm run dev
cd web
serverless deploy
```

## Accessing your Application
Domains will be dynamically constructed using the serverless stage, serverless service name and your domain, e.g. `https://dev-shopify-app-template.yourdomain.io`

Before attempting to test your application, you will need to update the app URLs in the Shopify Partner Dashboard, or you may run `npm run updateUrls` from the project root directory.

If you revert to local development/execution, the shopify app wrappers used by npm run dev should automatically reset the URL for you.  Note that its best practice to have multiple application definitions covering development, test & production use.
## Known Issues

### Image Rendering

.png issues may fail to render.  This is due to issues in serverless-http and/or API Gateway.  My current use-cases do not require images at this stage.

### serverless-offline cannot run functions backed by docker images

Refer to [this issue](https://github.com/dherault/serverless-offline/issues/1324).  Until [serverless-offline-lambda-docker-plugin](https://www.npmjs.com/package/serverless-offline-lambda-docker-plugin) supports NodeJS, the manual workaround is required to comment/uncomment sections of serverless.yaml.  The relevant sections are indicated with comments in the file.

### Session Handling

The shopify-app-template-node project includes the SQLite session store by default.  This session store is not useable if you attempt to build/bundle a zip file for lambda on an environment not compatible with the AWS runtime (Amazon Linux) as the binaries included will be for your development platform (in my case MacOS).
Further, the Lambdas execute with ephemeral storage for the container - changes to the database will not be persisted.
You WILL need to either

- change to a different database-backed session store.
- add a VPC and EFS volume to your serverless.yaml file, and update the sql database path in web/shopify.yaml to store the file on EFS.  SQLite is still not recommended for high volume production use.

Ideally, a DynamoDB implementation will become available shortly.

### Serverless Packaging Issues
Due to issues with the way Serverless wraps the function handler for deployment, I have not been able to successfully configure it, including with plugins such as serverless-webpack without encountering numerous issues with filename mappings, package.json type=module setting, and the issues with the inclusion of SQLLite binaries in a way which still allows the project to run in the default local development mode.
For this reason, the project is set up to build and run from ECR based docker images - this adds significant overhead to build & deployment time.
