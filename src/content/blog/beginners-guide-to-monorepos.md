---
title: "Beginner's guide to monorepos"
description: "Lately, we've been hearing more and more about monorepos. But what are monorepos, what are the differences between monorepos and poly-repos, and can you start using monorepos?"
pubDate: 'Jan 06 2025'
---

Lately, we've been hearing more and more about monorepos. But what are monorepos, what are the differences between monorepos and poly-repos, and can you start using monorepos?

A "poly-repo" or "multi-repo" architecture is the current standard for developing applications. Each team, project, library, service, etc., has its own repository. This separation creates a clear boundary for each component, giving each team autonomy to write and structure the code as they want and simplifying version control. However, this separation can add complexities, like fostering code duplication, "preventing" developers from collaborating between components/stacks, and inconsistencies between shared components.

In contrast, monorepos keep all projects and shared components in a single repository. This approach fosters tighter integration, simplifies code sharing, and enables a better developer workflow and mobility. However, the complexity of monorepos requires discipline to prevent pitfalls.

For this article, we will be using Turborepo by Vercel but there are multiple tools to work with monorepos. Turborepo works well with Javascript and Typescript codebases and it's perfect for NextJS apps.

### Monorepo structure

A monorepo is composed of apps and packages. For that, we have at the root of our monorepo a `apps` directory and a `packages` directory. In the root of our monorepo, we can also find a `package.json` file. In this file, we can find all the dependencies shared between all the apps and packages and the commands we can run in our mono-repo.

In the apps directory we have the apps of our monorepo. This can be apps for a backend and frontend for example.

The packages directory contains all the local packages that we create for our monorepo. These packages can be utils, UI components, shared configs, etc.

Inside each app, we can find a normal app like we find in any other project. All apps and packages need a `package.json` file with a name so we can run commands for that specific app or package as we will see later in the article.

In the root directory, we will also find another very important file for Turborepo monorepos. The `turbo.json` file lets Turborepo know all the commands we have in our monorepo, their dependencies and how they are supposed to be run.

### Create monorepo

To create a monorepo using Turborepo we can use the following command:

```bash
npx create-turbo@latest
```

Using this command we get two NextJS apps and UI, Eslint, and Typescript packages.

### Install NPM packages

To install packages in a monorepo we have to think of a different way of installing packages compared to multi-repos. In a monorepo, we have to decide if we want to install the package for a single app, a local package, or the whole mono-repo.

In most cases, we will want to install packages for a single app or package. To do that we can use the following command:

```bash
npm install zod --workspace my-app
```

If we want to install a package for the whole monorepo for example Eslint we can use the previous command but without specifying a workspace:

```bash
npm install zod
```

### Shared packages
While initially, I wanted to use the packages to share code from NPM packages, I found that this approach sometimes has some downsides.

This became obvious while configuring NextAuth in one mono repo. I wanted to create a package to share the configuration of NextAuth and export the NextAuth functions needed for the apps. While configuring the middleware, I had an error and couldn't figure out why. After some trial and error, I found that NextJS doesn't allow third-party export from packages.

The solution to this issue was to export only the NextAuth configuration from the `nextauth-config` package and install the NextAuth package directly in the apps.

To create packages we use `turbo gen workspace --name [package name] --type package`

To create a utils package we can use the command:

```bash
turbo gen workspace --name utils --type package
```

To add shared packages to an app you can add the app directly to the package.json of the app and then run the install command.

Example: Using the previous example of the Utils package, let's imagine we want to add the package to the web app. For that, we first start by adding the package to the package.json.

```json
{
  dependencies: {
    "utils": "*",
  }
}
```

With the dependency added we just have to run the install command to update our package-lock file:

```bash
npm install
```

### Deployment

To deploy projects with Turborepo we have to generate a 'clean' package.json for the app or package we want to build. To do this, we have to use the `prune` command provided by Turborepo.

To do this we can use the following command before the build step:

```bash
turbo prune api --docker
```

If you are using Docker to build your app you can use the same command

```docker
RUN turbo prune web --docker
```

Here is an example of a docker file using prune:

```docker
FROM node:18-alpine AS base

WORKDIR /app

RUN apk update
RUN apk add --no-cache libc6-compat
 
FROM base AS builder

RUN npm install -g turbo

COPY . .

# Here we build a prunned package.json and lockfile file for our web app
RUN turbo prune web --docker

FROM base AS installer

# Copy the prunned package.json and lockfile from the previous step
COPY --from=builder /app/out/json/ .
RUN npm install

COPY --from=builder /app/out/full/ .
RUN npm turbo run build
 
FROM base AS runner
WORKDIR /app

COPY --from=installer /app/apps/web/.next/standalone ./
COPY --from=installer /app/apps/web/.next/static ./apps/web/.next/static
COPY --from=installer /app/apps/web/public ./apps/web/public
 
CMD node apps/web/server.js
```

With this information, you have most of the knowledge necessary to start developing your apps using Turborepo.

For more information, you can check the official Turborepo documentation [here](https://turborepo.dev/docs).