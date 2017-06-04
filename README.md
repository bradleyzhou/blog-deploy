# blog-deploy
Deployment of blog.bradleyzhou.com using Docker containers


## Preparation
Start the preparation by cloning the git repo.
```
git clone https://github.com/bradleyzhou/blog-deploy.git
cd blog-deploy
```

### Build `blognginx` image
Prepare `frontend` static files
- Build the static files from [blog-frontend repo](https://github.com/bradleyzhou/blog-frontend) (e.g. run `npm run build`)
* Put these files under `nging/frontend` directory (e.g. rename and move `dist` directory)

This image exposes ports `80` and `443`
```
docker build -t blognginx nginx
```

### Build `blogapi` image
See [blog-api repo](https://github.com/bradleyzhou/blog-api)

### Tag and push docker images
```
docker tag blogapi <docker_username>/blogapi:<version>
docker tag blogapi <docker_username>/blogapi:latest
docker push <docker_username>/blogapi:<version>
docker push <docker_username>/blogapi:latest

docker tag blognginx <docker_username>/blognginx:<version>
docker tag blognginx <docker_username>/blognginx:latest
docker push <docker_username>/blognginx:<version>
docker push <docker_username>/blognginx:latest
```

## Deployment for development
### Prepare the docker images
On the deveopment machine, the images should be present since normally this is where the images were built. If not, just pull the images.

### Prepare Certs for SSL/HTTPS
Generate a pair of self-signed cert and key `./dev_cert.pem` and `dev_privkey.pem`. They are mapped in `dev.yml`
```
# generate a new pair of keys, answer questions accordingly when prompt
openssl req \
       -newkey rsa:2048 -nodes -keyout dev_privkey.pem \
       -x509 -days 365 -out dev_cert.pem
```

### Initialize the database
Database storage must be initialized before use. Neccessary environment variables are set in `dev.yml`, e.g. `PG_PASS`, `ADMIN_KEY`
```
# Create a brand new database
docker-compose -f docker-compose.yml -f dev.yml run db

# Wait for the database initialization and self-restart
# Then Ctrl-C to exit

# Init db table structures and add an admin user
docker-compose -f docker-compose.yml -f dev.yml -f dev-init.yml run blogapi
```

## Deployment for production
### Pull the docker images
On the production server, download the docker images
```
# for easier typing, assume root user
sudo -i

docker pull <docker_username>/blogapi
docker pull <docker_username>/blognginx
```

### Pull the deployment code
```
# for fresh deploy
mkdir /website && cd /website
git clone https://github.com/bradleyzhou/blog-deploy.git
cd blog-deploy

# if it is an update deploy
cd /website/blog-deploy
git pull
```

### Prepare the database
Data storage of Postgres image is mapped to `postgres/blogdata/`, and expects an empty directory. But to keep this directory in Git, a `.gitkeep` file is created (Git won't remember empty directories). To resolve this issue, delete the `.gitkeep` file.
```
rm ./postgres/blogdata/.gitkeep

# or create the dir if not exist
mkdir -p ./postgres/blogdata
```

When deploy for the first time, there won't be any database or schema. Thus for fresh deployment, the database must be initialized. Environment variables for creating database is specified in `.env` and `docker-compose.yml`.
```
# Create a brand new database, using settings in docker-compose.yml file
export PG_PASS=<choose_and_remember_db_pass>
docker-compose run db

# Wait for the database initialization and self-restart
# Then Ctrl-C to exit
```

Initialize the schemas and tables of the blog app. Then create a new admin for the blog. This user may well be the only user of the blog app. The commands are composed in `init.yml`.
```
# Init db table structures and add an admin user
export SECRET_KEY=<super_secret_key>
export PG_PASS=<db_pass>
export ADMIN_NAME=<your_name>
export ADMIN_EMAIL=<your@em.ail>
export ADMIN_KEY=<choose_and_remember_user_pass>
docker-compose -f docker-compose.yml -f init.yml run blogapi
```

## Run
```
# with console outputs
export SECRET_KEY=<super_secret_key>
export PG_PASS=<db_pass>
docker-compose up

# or in background
export SECRET_KEY=<super_secret_key>
export PG_PASS=<db_pass>
docker-compose up -d
```

## Notes on how it works
The configs are in `docker-compose.yml`, some environment variables are kept in `.env` which are automatically picked up by `docker-compose`. The config also servers as a documentation for the setup.

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
- The contents in `nginx/frontend` directory are copied into `blognginx` image at `/var/www` when building the image.
- Relavant configs are in `nginx/nginx.conf`, also built into `blognginx` image.

### Database
- Uses [`postgres` Docker image](https://hub.docker.com/_/postgres/), configured in `docker-compose.yml`.
- Database user, password, database name are set in `.env`, which is automatically loaded when using `docker-compose` commands.
- The actual storage of postgres database is mapped to directory `./postgre/blogdata`, so that the database won't disappear after container restart.
- The `blogapi` is instructed to use this database by environment variable settings in `docker-compose.yml`.
