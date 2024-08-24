FROM node:20

WORKDIR /app

ADD package*.json ./

RUN npm install

ADD prisma ./prisma

RUN npx prisma generate

ADD . .

EXPOSE 4000

RUN chmod +x startscript.sh