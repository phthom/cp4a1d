# Customize an existing Appsody stack

![image-20200406211104217](images/image-20200406211104217-6200264.png)

Developers use stacks to simplify building applications that require a specific set of technologies or development patterns. While there are numerous publicly available stacks to choose from, many enterprises want to build their own set of stacks that uphold their specific requirements and standards for how they want to their developers to build cloud native applications.

In this tutorial, you will learn how to modify an existing stack to more closely match your requirements. Before starting this, let’s do a quick review of the design requirements for stacks. A stack is designed to support the developer in either a rapid, local development mode or a build-and-deploy mode.

### Rapid, local development mode

In this mode, the stack contains everything a developer needs to build a new application on a local machine, with the application always running in a local containerized Docker environment. Introducing containerization from the start of the application development process (as opposed to development solely in the user space of the local machine) decreases the introduction of subtle errors in the containerization process and removes the need for a developer to install the core technology components of their application.

In this mode, the stack is required to have all the dependencies for the specific technologies pre-built into the Docker image, and also to dynamically compliment these with whatever dependencies the developer adds explicitly for his or her code.

Rapid local development mode in Appsody consists of the Appsody CLI (hooked into a local IDE if required) communicating with a local Docker container that is running the application under development. With this mode, application code can be held on the local file system, while being mounted in the Docker container, so that a local change can automatically trigger a restart of the application.



##1. Build-and-deploy mode

In this mode, the stack enables the Appsody CLI to build a self-contained Docker image that includes both the core technologies in the stack plus the application code, along with the combined dependencies of both. You can deploy the resulting image manually or programmatically to any platform that supports Docker images (such as a local or public Kubernetes cluster).

A pictorial view of how an application developer uses a stack, looks like this:

![Appsody Flow](https://developer.ibm.com/developer/tutorials/modify-appsody-stack/images/appsody.png)

The above development flow shows the manual deployment to a Kubernetes cluster. In more production-orientated environments, GitOps might trigger the build and deploy steps and Tekton Pipelines would drive the deployment. The [Kabanero](https://kabanero.io/) open source project brings together Appsody stacks, GitOps, and Tekton Pipelines into a single solution for cloud-native application development and deployment. [Kabanero Collections](https://github.com/kabanero-io/collections/), which is part of [IBM Cloud Pak for Applications](https://www.ibm.com/cloud/cloud-pak-for-applications), provides an enterprise-ready solution for cloud-native application development and deployment, based on these open source projects.

## 2. Understand the Appsody stack structure

Because a single Appsody stack can enable both rapid, local development and build-and-deploy modes, all stacks follow a standard structure. The structure below represents the source structure of a stack:

```ini
my-stack
├── README.md
├── stack.yaml
├── image/
|   ├── config/
|   |   └── app-deploy.yaml
|   ├── project/
|   |   ├── [files that provide the technology components of the stack]
|   |   └── Dockerfile
│   ├── Dockerfile-stack
|   └── LICENSE
└── templates/
    ├── my-template-1/
    |       └── [example files as a starter for the application, e.g. "hello world"]
    └── my-template-2/
            └── [example files as a starter for a more complex application]
```

As a *stack architect* you must create the above structure, build it into an actual stack image ready for use by an *application developer* who bases their new application on your stack. Part of your role as a stack architect is to include one of more sample applications (known as *templates*) to help the application developer get started.

Hence, when you build a stack, the structure above is processed and generates a Docker image for the stack, along with tar files of each of the templates, which can then all be stored and referenced in a local or public Appsody repo. The Appsody CLI can access the repo to use the stack to initiate local development.

In this tutorial, we show you how to modify the existing `nodejs-express` stack, that is available in the main Appsody repo, to add some additional security hardening. Individual enterprises often have specific security standards that need to be met to allow deployment.

> **NOTE**: For future reference, to make your own stack from scratch instead of extending an existing one, follow [this tutorial](https://developer.ibm.com/tutorials/create-appsody-stack/).

## 3. Create a new stack, based on an existing one

The goal of this step is to create a new Node.js Express stack by modifying the existing one. We’ll copy it, build, and modify it.

### Initialize the stack

To create a new stack, you must first construct a scaffold of the above structure. Stacks are classified as being `stable`, `incubating` or `experimental`. You can read more about these classifications [here](https://appsody.dev/docs/stacks/stacks-overview). To make things easy, the Appsody CLI supports an `appsody stack create` command to create a new stack, by copying an existing one.

When you run the `appsody stack create` command, the *nodejs-express* stack is copied and moved, and a directory is created that contains the new stack.

```bash
cd ~
cd my	
cd my-nodejs-express
ls -al
```

You should see output similar to the following:

```bash
$ ls -la
total 24
drwxr-xr-x  6 henrynash  staff   192  8 Nov 11:51 .
drwxr-xr-x  5 henrynash  staff   160  8 Nov 11:51 ..
-rw-r--r--  1 henrynash  staff  5026  8 Nov 11:51 README.md
drwxr-xr-x  7 henrynash  staff   224  8 Nov 11:51 image
-rw-r--r--  1 henrynash  staff   319  8 Nov 11:51 stack.yaml
drwxr-xr-x  4 henrynash  staff   128  8 Nov 11:51 templates
```

If you inspect the contents of the `image` directory, you will see how it matches the stack structure given earlier.

### Build your new stack

Before we make any changes, let’s go through the steps of building (or *packaging*) a stack, to create a stack image (which is a Docker image) that the Appsody CLI can use to initiate a project using that stack.

There is a Docker file (`Dockerfile-stack`) within the sample stack structure you copied. The `appsody stack package` command uses this to build the image.

To build your new stack in this way, from the `my-nodejs-express` directory enter:

```bash
appsody stack package
```

This runs a Docker build, installs `my-nodejs-express` into a local Appsody repository (called `dev.local`), and runs some basic tests to make sure the file is well formed.

Once the build is complete, use the `appsody list` command to check that it is now available in the local repo:

```bash
appsody list dev.local
```

You should see output similar to the following:

```bash
$ appsody list dev.local

REPO     	ID               	VERSION  	TEMPLATES        	DESCRIPTION                      
dev.local	my-nodejs-express	0.4.6    	scaffold, *simple	Express web framework for Node.js
```



### Run the new stack

So, at this point, you have been carrying out your role as a stack architect to build and install your new (albeit unchanged) stack. Now it’s time to try it out as an application developer.

Create a new directory and initialize it with this new Appsody stack:

```bash
mkdir ~/test-my-stack
cd ~/test-my-stack
appsody init dev.local/my-nodejs-express
```

Now use `appsody run` to test running an application based on your copy of the stack:

```bash
appsody run
```

You should see output similar to the following:

```bash
$ appsody run
Running development environment...
Using local cache for image dev.local/my-nodejs-express:SNAPSHOT
Running docker command: docker run --rm -p 3000:3000 -p 9229:9229 --name test73-dev -v /Users/henrynash/codewind-workspace/test73/:/project/user-app -v test73-deps:/project/user-app/node_modules -v /Users/henrynash/.appsody/appsody-controller:/appsody/appsody-controller -t --entrypoint /appsody/appsody-controller dev.local/my-nodejs-express:SNAPSHOT --mode=run
[Container] Running APPSODY_PREP command: npm install --prefix user-app && npm audit fix --prefix user-app
added 170 packages from 578 contributors and audited 295 packages in 3.5s
[Container] found 0 vulnerabilities
...
[Container] App started on PORT 3000
```

To check it is running, in a separate terminal window you can use `curl` to hit the endpoint:

```bash
curl -v localhost:3000
$ curl -v localhost:3000
Hello from Appsody!
```

So now we are ready to make changes to our new stack. In this tutorial, we harden the HTTP headers that an application, built using this stack, responds with. We can look at the current headers returned:

```bash
$ curl -v localhost:3000
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 3000 (#0)
> GET / HTTP/1.1
> Host: localhost:3000
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 200 OK
< X-Powered-By: Express
< Content-Type: text/html; charset=utf-8
< Content-Length: 19
< ETag: W/"13-0ErcqB22cNteJ3vXrBgUhlCj8os"
< Date: Mon, 21 Oct 2019 12:09:49 GMT
< Connection: keep-alive
<
* Connection #0 to host localhost left intact
Hello from Appsody!
```

Stop this current appsody run by either using `CNTL-C` in the initial terminal window or running `appsody stop` in a separate terminal window from within the same directory.

In this tutorial, we show you how to modify the stack to include the popular HTTP header security module [helmet](https://helmetjs.github.io/), which should change the headers that are returned to us. Note we will do this as a *stack architect* since we don’t want to rely on *application developers* remembering to do this. By doing this in the stack itself, all applications built using our modified stack will have helmet automatically enabled.

### Modify your custom stack

When creating a custom stack, based on an existing stack, the first thing to do is to take a look at what the existing stack provides. A more detailed description of the stack components can be found [here](https://appsody.dev/docs/stacks/stack-structure), but the key ones are:

- A Dockerfile (`image/Dockerfile-stack`) that builds your stack image. This is what the `appsody stack package` command used above to build a Docker image of your stack — this is the eventual artifact that you deliver as a stack architect to application developers.
- A Dockerfile (`image/project/Dockerfile`) that builds the final application image. This final image contains both your stack and its application, and this Dockerfile is processed by the application developer running `appsody build` and `appsody deploy`.
- Typically some kind of server-side code that enables the application the developer will create and run. For this stack, this code is `image/project/server.js`.
- Some kind of dependency management, ensuring both the correct inclusion of components defined by the stack, as well as, potentially, any added by the application developer. For this stack, this is `image/project/package.json`.
- At least one sample application (or *template*); these are stored in the `templates` directory.

You should take some time to check out the files given above to get a feel of the stack.

For some stack modifications, you can actually use a form of stack inheritance — that is, by using the existing stack images as the `FROM` image in `Dockerfile-stack`. An example of this might be where you just want to change one of the Dockerfile variables. In general, however, most modified stacks are effectively copies of an existing stack, with the additional changes added to gain the new, required functionality.

Having examined the files above, you might have already spotted what we need to do to incorporate helmet into the new stack — namely to modify `image/project/server.js` to enable it.

Go back to the `my-nodejs-express` directory.

```bash
cd ~/my-nodejs-express
```

The current code in `image/project/server.js` looks something like this:

```javascript
// Requires statements and code for non-production mode usage
if (!process.env.NODE_ENV || !process.env.NODE_ENV === 'production') {
require('appmetrics-dash').attach();
}
const express = require('express');
const health = require('@cloudnative/health-connect');
const fs = require('fs');

require('appmetrics-prometheus').attach();

const app = express();

const basePath = __dirname + '/user-app/';

function getEntryPoint() {
    let rawPackage = fs.readFileSync(basePath + 'package.json');
    let package = JSON.parse(rawPackage);
    if (!package.main) {
        console.error("Please define a primary entrypoint of your application by adding 'main: <entrypoint>' to package.json.")
        process.exit(1)
    }
    return package.main;
}

// Register the users app. As this is before the health/live/ready routes,
// those can be overridden by the user
const userApp = require(basePath + getEntryPoint()).app;
app.use('/', userApp);

const healthcheck = new health.HealthChecker();
app.use('/live', health.LivenessEndpoint(healthcheck));
app.use('/ready', health.ReadinessEndpoint(healthcheck));
app.use('/health', health.HealthEndpoint(healthcheck));

app.get('*', (req, res) => {
res.status(404).send("Not Found");
});

const PORT = process.env.PORT || 3000;
const server = app.listen(PORT, () => {
console.log(`App started on PORT ${PORT}`);
});

// Export server for testing purposes
module.exports.server = server;
module.exports.PORT = PORT;
```

Modify this file by adding two lines: one line to import helmet (with `require()`) and one line to enable it with `app.use()`:

```javascript
// Requires statements and code for non-production mode usage
if (!process.env.NODE_ENV || !process.env.NODE_ENV === 'production') {
require('appmetrics-dash').attach();
}
const express = require('express');
const helmet = require('helmet');
const health = require('@cloudnative/health-connect');
const fs = require('fs');

require('appmetrics-prometheus').attach();

const app = express();
app.use(helmet());

const basePath = __dirname + '/user-app/';
...
```

Since you added a new required module, you must also update the dependency management (`package.json`), to ensure this is pulled in:

```json
{
...
"dependencies": {
    "@cloudnative/health-connect": "^2.0.0",
    "appmetrics-prometheus": "^3.0.0",
    "express": "~4.16.0",
    "helmet": "^3.21.1"
},
...
}
```

Finally, we should change the description of our stack, so that application developers can distinguish it from the original nodejs-express stack. The meta data of a stack is stored in a file called `stack.yaml` in the top-level directory of the stack.

You can look at the current contents of `stack.yaml` like this:

```bash
$ cat ~/stack.yaml
name: Node.js Express
version: 0.2.8
description: Express web framework for Node.js
license: Apache-2.0
language: nodejs
maintainers:
  - name: Chris Bailey
    email: cnbailey@gmail.com
    github-id: seabaylea
  - name: Neeraj Laad
    email: neeraj.laad@gmail.com
    github-id: neeraj-laad
default-template: simple
```

You should edit this file to change the description to something more descriptive like, “Secure express web framework for Node.js”. Typically, you would also make yourself the maintainer and decide on a version numbering scheme.

Now that we’ve modified our stack, we need to re-package it, using the same command as before:

```bash
appsody stack package
```

This will update the dev.local index, which you can again list:

```bash
appsody list dev.local
```

You should see output similar to the following:

```bash
$ appsody list dev.local
REPO        ID                  VERSION     TEMPLATES           DESCRIPTION
dev.local   my-nodejs-express   0.2.8       scaffold, *simple   Secure express web framework for Node.js
```

To test this out, run your application:

```bash
cd ~/test-my-stack
appsody run
```

If you hit the endpoint as before with `curl` in verbose mode, you can see if the HTTP headers have changed:

```bash
curl -v localhost:3000
```

You should now see security-related headers like `X-DNS-Prefetch-Control`, `Strict-Transport-Security`, and `X-Download-Options`:

```bash
$ curl -v localhost:3000
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 3000 (#0)
> GET / HTTP/1.1
> Host: localhost:3000
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 200 OK
< X-DNS-Prefetch-Control: off
< X-Frame-Options: SAMEORIGIN
< Strict-Transport-Security: max-age=15552000; includeSubDomains
< X-Download-Options: noopen
< X-Content-Type-Options: nosniff
< X-XSS-Protection: 1; mode=block
< X-Powered-By: Express
< Content-Type: text/html; charset=utf-8
< Content-Length: 19
< ETag: W/"13-0ErcqB22cNteJ3vXrBgUhlCj8os"
< Date: Fri, 08 Nov 2019 19:39:22 GMT
< Connection: keep-alive
<
* Connection #0 to host localhost left intact
Hello from Appsody!
```

As you should see, because the stack now incorporates helmet, the HTTP headers have changed and our application runs with this protection. The inclusion of helmet is just an example of some of the security hardening you might want to add within your own enterprise.

## Summary

**Congratulations**! You have successfully built and tested a modified stack — and seen how applications built against the stack automatically gain the (new) features it provides (without the application developer having to do anything themselves). You might just want to use this stack within our enterprise – or alternatively consider submitting your new stack to the Appsody open source project if you think it would be useful to application developers more broadly.

- 