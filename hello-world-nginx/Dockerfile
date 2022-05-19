ARG NGINX_IMG_TAG=alpine
FROM nginx:${NGINX_IMG_TAG}

COPY docker-deps/default.conf /etc/nginx/conf.d/default.conf

RUN /bin/sed -i -e '/^worker_processes/s/auto/1/' /etc/nginx/nginx.conf
