FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . ./

RUN npm uninstall esbuild
RUN npm install esbuild

RUN npm run build

COPY ./build ./build

ENV HOST=0.0.0.0
EXPOSE 80 3000

CMD ["node", "build"]