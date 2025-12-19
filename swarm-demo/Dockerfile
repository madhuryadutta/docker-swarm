FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.template.html
COPY default.conf /etc/nginx/conf.d/default.conf

CMD sh -c "envsubst < /usr/share/nginx/html/index.template.html > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'"
