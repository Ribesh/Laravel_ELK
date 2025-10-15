
#Stage 1: Build Laravel frontend assets
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Nginx production image
FROM nginx:alpine

WORKDIR	/var/www/public

COPY --from=builder /app/public/build /var/www/public/build
COPY ./public /var/www/public
COPY ./nginx/conf.d/app.conf /etc/nginx/conf.d/default.conf

RUN chown -R nginx:nginx /var/www/public
RUN chown -R 777 /var/www/public
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]


