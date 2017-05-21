# blog-deploy
Deployment of blog.bradleyzhou.com


## Build blognginx image
```
docker build -t blognginx nginx
```

## Run the containers for the blog app
### Preparation
docker images:
* blogapi, from [blogapi repo][https://github.com/bradleyzhou/blog-api]
* blognginx, from `nginx` directory here

### Run
```
docker compose up
```

## Notes on how it works
In docker image `blogapi`, uWSGI runs with a socket file `/BlogAPISock/app.sock`, file owner is `www-data`.
`/BlogAPISock` is mounted as a volume named `blogapi-sock`.
The volume `blogapi-sock` is also mounted when image `blognginx` runs.
In image `blogapi-sock`, Nginx is configured to run as user `www-data`, and communicate to `/BlogAPISock/app.sock`.

```
                  blognginx                 Volume              blogapi
Incoming     ---------------------     ----------------     ----------------
Requests <-> | nginx unix socket | <-> | blogapi-sock | <-> | uWSGI socket |
             ---------------------     ----------------     ----------------
```
