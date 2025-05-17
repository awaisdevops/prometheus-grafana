FROM node:13-alpine

RUN mkdir -p /usr/app
COPY app/images/* /usr/app/images/
COPY app/index.html /usr/app/
COPY app/package.json /usr/app/
COPY app/server.js /usr/app/

WORKDIR /usr/app
EXPOSE 3000

RUN npm install

CMD ["node", "server.js"]