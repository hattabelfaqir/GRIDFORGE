FROM node:22-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

CMD ["npx", "@tailwindcss/cli", "-i", "./src/input.css", "-o", "/output/style.css", "--watch"]
