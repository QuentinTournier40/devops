FROM node:18-slim

WORKDIR /usr/local/app

COPY . .

RUN npm install -g nodemon && \
    npm ci && \
    npm cache clean --force && \
    mv /usr/local/app/node_modules /node_modules && \
    rm package.json && \
    rm package-lock.json

ENV PORT=4000

EXPOSE $PORT

CMD [ "node", "server.js" ]