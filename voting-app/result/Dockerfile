FROM node:17-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
EXPOSE 8888  
CMD ["npm", "start"]