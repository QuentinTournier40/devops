FROM node:18-slim

WORKDIR /usr/local/app

COPY package*.json .

RUN npm install -g nodemon && \
    npm ci && \
    npm cache clean --force && \
    mv /usr/local/app/node_modules /node_modules

COPY . .

ENV PORT=4000

EXPOSE $PORT

CMD [ "node", "server.js" ]