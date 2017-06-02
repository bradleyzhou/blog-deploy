FROM nginx

EXPOSE 80 443

ADD ./nginx.conf /etc/nginx/
ADD ./frontend /var/www/blog

STOPSIGNAL SIGQUIT

CMD ["nginx", "-g", "daemon off;"]
