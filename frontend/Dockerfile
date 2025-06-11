FROM node:21 AS builder

WORKDIR /app

COPY package*.json ./

RUN npm i

COPY . .

FROM node:21-slim

WORKDIR /app

COPY --from=builder /app ./

COPY .env.sample .env.local

CMD ["npm", "run", "dev","--", "--host"]

