FROM nginx

RUN mkdir /BlogAPISock
ADD nginx.conf /etc/nginx/

EXPOSE 80 443

STOPSIGNAL SIGQUIT

CMD ["nginx", "-g", "daemon off;"]
