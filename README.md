# FastAPI Example Project
Some people were searching my GitHub profile for project examples after reading the article on [FastAPI best practices](https://github.com/zhanymkanov/fastapi-best-practices).
Unfortunately, I didn't have useful public repositories, but only my old proof-of-concept projects. 

Hence, I have decided to fix that and show how I start projects nowadays, after getting some real-world experience. 
This repo is kind of a template I use when starting up new FastAPI projects:
- some configs for production
  - gunicorn with dynamic workers configuration (stolen from [@tiangolo](https://github.com/tiangolo))
  - Dockerfile optimized for small size and fast builds with a non-root user
  - JSON logs
  - sentry for deployed envs
- easy local development
  - environment with configured postgres and redis
  - script to lint code with `ruff` and `ruff format`
  - configured pytest with `async-asgi-testclient`, `pytest-env`, `pytest-asyncio`
- SQLAlchemy with slightly configured `alembic`
  - async SQLAlchemy engine
  - migrations set in easy to sort format (`YYYY-MM-DD_slug`)
- pre-installed JWT authorization
  - short-lived access token
  - long-lived refresh token which is stored in http-only cookies
  - salted password storage with `bcrypt`
- global pydantic model with 
  - explicit timezone setting during JSON export
- and some other extras like global exceptions, sqlalchemy keys naming convention, shortcut scripts for alembic, etc.

Current version of the template (with SQLAlchemy >2.0 & Pydantic >2.0) wasn't battle tested on production, 
so there might be some workarounds instead of neat solutions, but overall idea of the project structure is still the same.

## Local Development (with docker)

### First Build Only
1. `cp .env.example .env` and edit .env 
2. `source .env` or reopen terminal
3. `docker network create app_net`
4. `docker-compose up -d --build`

### Linters
Format the code with `ruff --fix` and `ruff format`
```shell
docker compose exec app format
```

### Migrations
- Create an automatic migration from changes in `src/database.py`
```shell
docker compose exec app makemigrations *migration_name*
```
- Run migrations
```shell
docker compose exec app migrate
```
- Downgrade migrations
```shell
docker compose exec app downgrade -1  # or -2 or base or hash of the migration
```
### Tests
All tests are integrational and require DB connection. 

One of the choices I've made is to use default database (`postgres`), separated from app's `app` database.
- Using default database makes it easier to run tests in CI/CD environments, since there is no need to setup additional databases
- Tests are run with upgrading & downgrading alembic migrations. It's not perfect, but works fine. 

Run tests
```shell
docker compose exec app pytest
```
### Justfile
The template is using [Just](https://github.com/casey/just). 

It's a Makefile alternative written in Rust with a nice syntax.

You can find all the shortcuts in `justfile` or run the following command to list them all:
```shell
just --list
```
Info about installation can be found [here](https://github.com/casey/just#packages).
### Backup and Restore database
We are using `pg_dump` and `pg_restore` to backup and restore the database.
- Backup
```shell
just backup
# output example
Backup process started.
Backup has been created and saved to /backups/backup-year-month-date-HHMMSS.dump.gz
```

- Copy the backup file or a directory with all backups to your local machine
```shell
just mount-docker-backup  # get all backups
just mount-docker-backup backup-year-month-date-HHMMSS.dump.gz  # get a specific backup
```
- Restore
```shell
just restore backup-year-month-date-HHMMSS.dump.gz
# output example
Dropping the database...
Creating a new database...
Applying the backup to the new database...
Backup applied successfully.
```

### Debugging in VSCode

[How to debug Python apps inside a Docker Container with VS Code](https://www.python-engineer.com/posts/debug-python-docker/)

- attach debugger with Python: Remote Attach in launch.json

## Local Development and Debugging (with pipenv)

### docker services excluding only app
1. `cp .env.example .env` and edit .env 
2. `source .env` or reopen terminal
3. `docker network create app_net`
4. `docker-compose up -d app_db app_db_admin app_redis`


### install pipenv and dependencies (packages are installed in the virtual environment, not globally)
```bash
# Install packages from Pipfile
pipenv install --dev 
pipenv shell

# Database migration (create tables)
alembic upgrade head
```

### Debuggin in VSCode with pipenv

  - VSCode 하단의 상태 바에서 Python 인터프리터 버전을 클릭합니다. (python 파일 편집 상태에서 우측 하단) 
  - 만약 Pipenv 환경이 보이지 않는다면, Ctrl+Shift+P (Windows/Linux) 또는 Cmd+Shift+P (Mac), Python: Select Interpreter