# blog-deploy
Deployment of blog.bradleyzhou.com


## Build blognginx image
```
docker build -t blognginx nginx
```

## Run the containers for the blog app
### Preparation
- docker images:
    * blogapi, from [blog-api repo](https://github.com/bradleyzhou/blog-api)
    * blognginx, from `nginx` directory here
- frontend static files under `frontend` directory, built from [blog-frontend repo](https://github.com/bradleyzhou/blog-frontend)

### Run
```
# with console outputs
docker-compose up

# in background
docker-compose up -d
```

## Notes on how it works
### Socket communication of `uwsgi` and `nginx`
- In docker image `blogapi`, uWSGI runs with a socket file `/BlogAPISock/app.sock`, file owner is `www-data`.
- `/BlogAPISock` is mounted as a volume named `blogapi-sock`.
- The volume `blogapi-sock` is also mounted when image `blognginx` runs.
- In image `blogapi-sock`, Nginx is configured to run as user `www-data`, and communicate to `/BlogAPISock/app.sock`.

```
                  blognginx                 Volume              blogapi
Incoming     ---------------------     ----------------     ----------------
Requests <-> | nginx unix socket | <-> | blogapi-sock | <-> | uWSGI socket |
             ---------------------     ----------------     ----------------
```

### Frontend static files
- The frontend static files are built by `npm build` in [blog-frontend repo](https://github.com/bradleyzhou/blog-frontend)
- The build will output a `dist` directory, which is renamed `frontend` and moved to current directory
- The `frontend` directory is mapped into `blognginx` image in `docker-compose.yml`
- Relavant configs are in `nginx/nginx.conf`, also mapped into `blognginx` image
- When `blognginx` image runs, it will find the correct image and static files
