FROM node:18-alpine
WORKDIR /app/authService
COPY authService/package*.json ./
RUN npm install --production
COPY authService/. ./
COPY common ../common
EXPOSE 3001
CMD ["npm", "run", "start"]