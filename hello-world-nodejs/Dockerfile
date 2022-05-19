ARG NODE_BUILD_IMG_TAG=16
ARG NODE_RUN_IMG_TAG=alpine
FROM node:${NODE_BUILD_IMG_TAG} AS npm-install

COPY . /app 
WORKDIR /app 

RUN npm install

FROM node:${NODE_RUN_IMG_TAG}

COPY --from=npm-install --chown=node:node /app /app
USER node:node
WORKDIR /app

CMD ["npm", "start"]
