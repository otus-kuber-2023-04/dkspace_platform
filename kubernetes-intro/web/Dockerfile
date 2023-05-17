FROM nginx:latest
WORKDIR /app
RUN apt-get update -y && apt-get install -y wget mc && usermod -u 1001 -g 101 nginx
COPY app/ ./
COPY default.conf /etc/nginx/conf.d/

