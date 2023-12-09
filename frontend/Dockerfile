FROM node:18 AS builder
RUN curl -o- -L https://yarnpkg.com/install.sh | bash

# set working directory

ENV WKDIR /opt/react_app
WORKDIR ${WKDIR}

COPY package*.json .

RUN npm ci

ENV PATH ${WKDIR}/node_modules/.bin:$PATH

COPY . .

CMD ["npm", "start"]