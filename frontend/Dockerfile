FROM node:18
WORKDIR /app
COPY . .
RUN npm install && npm run build

FROM nginx:latest
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist/frontend /usr/share/nginx/html
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]

