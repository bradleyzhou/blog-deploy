# blog-deploy
Deployment of blog.bradleyzhou.com using Docker containers


## Preparation
Start by cloning the git repo.
```
git clone https://github.com/bradleyzhou/blog-deploy.git
cd blog-deploy
```

### Build `blognginx` image
This modified image `blognginx` exposes ports `80` and `443`
```
docker build -t blognginx nginx
```

### Build `blogapi` image
See [blog-api repo](https://github.com/bradleyzhou/blog-api)

### Prepare `frontend` static files
- Build the static files from [blog-frontend repo](https://github.com/bradleyzhou/blog-frontend) (e.g. run `npm run build`)
- Put these files under `frontend` directory (e.g. rename and move `dist` directory)

### Prepare the database
Data storage of Postgres image is mapped to `postgres/blogdata/`, and expects an empty directory. But to keep this directory in Git, a `.gitkeep` file is created (Git won't remember empty directories). To resolve this issue, delete the `.gitkeep` file.
```
rm ./postgres/blogdata/.gitkeep
```

When deploy for the first time, there won't be any database or schema. Thus for fresh deployment, the database must be initialized. Environment variables for creating database is specified in `.env` and `docker-compose.yml`.
```
# Create a brand new database, using settings in docker-compose.yml file
PG_PASS=<choose_and_remember_db_pass> docker-compose run db

# Wait for the database initialization and self-restart
# Then Ctrl-C to exit
```

Initialize the schemas and tables of the blog app. Then create a new admin for the blog. This user may well be the only user of the blog app. The commands are composed in `init.yml`.
```
# Init db and admin user
PG_PASS=<db_pass> \
ADMIN_NAME=<your_name> \
ADMIN_EMAIL=<your@em.ail> \
ADMIN_KEY=<choose_and_remember_user_pass> \
docker-compose -f docker-compose.yml -f init.yml run blogapi
```

## Run
```
# with console outputs
docker-compose up

# in background
docker-compose up -d
```

## Notes on how it works
The configs are in `docker-compose.yml`. It also servers as a documentation for the setup.

### Socket communication of `uwsgi` and `nginx`
- In docker image `blogapi`, uWSGI runs with a socket file `/BlogAPISock/app.sock`, file owner is `www-data`.
- `/BlogAPISock` is mounted as a volume named `blogapi-sock`.
- The volume `blogapi-sock` is also mounted when image `blognginx` runs.
- In image `blogapi-sock`, Nginx is configured to run as user `www-data`, and communicate to `/BlogAPISock/app.sock`.

```
                  blognginx            blogapi-sock         blogapi
Incoming     ---------------------     ------------     ----------------
Requests <-> | nginx unix socket | <-> | app.sock | <-> | uWSGI socket |
             ---------------------     ------------     ----------------
```

### Frontend static files
- The frontend static files are built by `npm build` in [blog-frontend repo](https://github.com/bradleyzhou/blog-frontend).
- The build will output a `dist` directory, which is renamed `frontend` and moved to current directory.
- The `frontend` directory is mapped into `blognginx` image in `docker-compose.yml`.
- Relavant configs are in `nginx/nginx.conf`, also mapped into `blognginx` image.
- When `blognginx` image runs, it will find the correct image and static files.

### Database
- Uses [`postgres` Docker image](https://hub.docker.com/_/postgres/), configured in `docker-compose.yml`.
- Database user, password, database name are set in `.env`, which is automatically loaded when using `docker-compose` commands.
- The actual storage of postgres database is mapped to directory `./postgre/blogdata`, so that the database won't disappear after container restart.
- The `blogapi` is instructed to use this database by environment variable settings in `docker-compose.yml`.
