---
layout: post
title:  "Using environment variables with single page apps and static websites"
categories: post
---

When creating a frontend environment variables are used to configure how the apps work. When it comes time to deploy, the application is built and the output is moved to a web server. But, what if there needs to be a difference in how the application functions based on its environment.

Say there is a QA environment and a production environment. There shouldn't be a seperate build to handle the differents environment's client IDs, backend addresses, or feature flags. 

This is how I solved this problem with my React app. Allowing me to set environment variables, and have those variables sent to the user's browser without having to re-build the singe page app (SPA).
I am building with Vite.

## 1. Define a vite config that seperates the environment variables into a seperate file.

```
export default defineConfig({
    plugins: [react()],
    build: {
        rollupOptions: {
            output: {
                manualChunks(id) {
                    if (/envVariables.ts/.test(id)) {
                        return 'envVariables'
                    }
                },
            },
        }
    }
})
```

This will cause the `envVariables` file (which will be made in a second) to be built to a seperate file from all the rest of the app's output.

## 2. Create an `envVariables.ts` file that the app will use to get its configuration.

Create `envVariables.ts` file in the src folder with the config.
```
const config = {
    VITE_NAME: '$VITE_NAME',
}

export const envVariables = {
    VITE_NAME: !config.VITE_NAME.includes('VITE_') ? config.VITE_NAME : import.meta.env.VITE_NAME,
};
```

What this will do is provide a way to still use .env files when developing locally but also give a way for a web server to do a find-and-replace of the environment variables.

Use it like so:

```
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { envVariables } from './envVariables';

createRoot(document.getElementById('root')!).render(
    <StrictMode>
        <div>Hello {envVariables.VITE_NAME}</div>
    </StrictMode>
);
```

## 3. Dockerize the application by building off of an Nginx image.

This is an example Dockerfile that will build the application and serve it using Nginx.

```
FROM node:20-alpine AS builder

WORKDIR /app
COPY package.json .
COPY package-lock.json .

RUN npm ci

COPY . .

RUN npm run build

FROM nginx:1.27-alpine AS production

EXPOSE 80

COPY .docker/100-env-variables.sh /docker-entrypoint.d/
RUN chmod +x /docker-entrypoint.d/100-env-variables.sh
COPY .docker/default.conf /etc/nginx/conf.d/

WORKDIR /usr/share/nginx/html

COPY --from=builder /app/dist ./
```

This assumes that the output of `npm run build` will output to the `dist` folder.
Notice the two files that are copied to the image. Those will be covered in the next step.

## 3. Create scripts that nginx will run on container startup.

1. Create `.docker/100-env-variables.sh` with the following content:

```
#!/bin/sh

# this file has to use LF line-endings
envVariables=$(ls -t /usr/share/nginx/html/assets/envVariables-*.js | head -n1)
envsubst < "$envVariables" > ./envVariablesTemp
mv ./envVariablesTemp "$envVariables"
```

This file will run when the container starts up. It will replace all the '$VITE_*' strings in `envVariables` with the matching environment variables. 

2. Create `.docker/default.conf` for your nginx config

Alternatively hash routing could be used but this will stop Nginx from returning a 404 for any url that isn't the root.

```
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        # this line will send the static site instead of the generic 404 page.
        try_files $uri /index.html =404;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

## 4. Create the docker image and try it out.

`docker build -t env-spa .`\
`docker run -p 8080:80 -e VITE_NAME=Labod env-spa`