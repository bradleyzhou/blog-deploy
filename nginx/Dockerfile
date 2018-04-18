FROM nginx

EXPOSE 80 443

ARG blog_frontend_release
ARG release_url=https://github.com/bradleyzhou/blog-frontend/releases/download/${blog_frontend_release}/blog-frontend-${blog_frontend_release}.tar.gz

ADD ./nginx.conf /etc/nginx/
ADD ${release_url} dist.tar.gz
RUN mkdir -p /var/www && tar xvzf dist.tar.gz -C /var/www/
RUN mv /var/www/dist /var/www/blog

STOPSIGNAL SIGQUIT

CMD ["nginx", "-g", "daemon off;"]
