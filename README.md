# VNOJ Docker

A Docker-based setup for [VNOJ](https://github.com/VNOI-Admin/OJ) (Vietnam Online Judge), based on [dmoj-docker](https://github.com/Ninjaclasher/dmoj-docker).

## Features

- **Full VNOJ site** with Nginx, uWSGI, MySQL, Redis
- **Judge server** with Themis support (C++, Pascal)
- **20+ programming languages** supported
- **Vietnamese/English** interface with i18n
- **PDF problem statements** with embedded viewer
- **Contest management** with multiple format support

## Quick Start

### Prerequisites

- Docker & Docker Compose (v2+)
- Git

### Setup

```sh
# Clone with submodules
git clone --recursive https://github.com/hd929/vnoj-docker.git
cd vnoj-docker/dmoj

# Initialize config
./scripts/initialize

# Configure environment
cp environment/mysql-admin.env.example environment/mysql-admin.env
cp environment/mysql.env.example environment/mysql.env
cp environment/site.env.example environment/site.env
# Edit the .env files to set passwords and secret key
cp ../config/local_settings.py repo/dmoj/

# Build and start
docker compose build
docker compose up -d

# Run migrations
docker exec vnoj_site python3 manage.py migrate

# Load fixtures
docker exec vnoj_site python3 manage.py loaddata navbar
docker exec vnoj_site python3 manage.py loaddata language_small
docker exec vnoj_site python3 manage.py loaddata demo

# Collect static files
docker exec vnoj_site python3 manage.py collectstatic --noinput

# Create admin
docker exec vnoj_site python3 manage.py createsuperuser
```

Access at **http://localhost** (admin: `/admin/`)

### Running the Judge

```sh
# Create judge account
docker exec vnoj_site python3 manage.py shell -c "
from judge.models import Judge
Judge.objects.get_or_create(name='vnoj-judge', defaults={'auth_key': 'your_key'})
"

# Start judge container
docker run -d --name vnoj_judge \
  --restart unless-stopped \
  --network dmoj_db \
  --cap-add=SYS_PTRACE \
  -v /path/to/problems:/problems \
  -v /path/to/judge.yml:/judge.yml \
  vnoj/judge-tiervnoj:latest \
  run -c /judge.yml bridged vnoj-judge your_key
```

## Architecture

| Container | Role |
|-----------|------|
| `vnoj_mysql` | MariaDB 10.11 database |
| `vnoj_redis` | Redis cache & queue |
| `vnoj_site` | Django uWSGI application |
| `vnoj_nginx` | Nginx reverse proxy |
| `vnoj_bridged` | Judge bridge daemon |
| `vnoj_celery` | Background task worker |
| `vnoj_wsevent` | WebSocket event server |
| `vnoj_judge` | Submission grader |

## Customization

### UI / Branding

- **Logo**: Replace `resources/icons/logo.svg` and `resources/icons/logo.png`
- **Favicon**: Replace `resources/favicon.ico`
- **Theme colors**: Edit `resources/vars-default.scss` (light) and `resources/vars-dark.scss` (dark)
- **Site name**: Set `SITE_NAME` and `SITE_LONG_NAME` in `config/local_settings.py`

### Language / i18n

- Translations are in `repo/locale/vi/LC_MESSAGES/django.po`
- Compile with: `docker exec vnoj_site python3 manage.py compilemessages`

### Features

- **Pwned password check**: Toggle with `DMOJ_ENABLE_PWNED_PASSWORD_CHECK` in `config/local_settings.py`
- **Password validators**: Override `AUTH_PASSWORD_VALIDATORS` in `local_settings.py` to remove checks

## Adding Problems

### Via Admin Interface

1. Go to **http://localhost/admin/judge/problem/add/**
2. Upload test data as ZIP or add test cases manually
3. Set `pdf_url` to link a PDF statement (e.g., `/pdf/problems/mystatement.pdf`)

### Via Script

```python
from judge.models import Problem, ProblemTestCase
prob = Problem.objects.create(code='myprob', name='My Problem', ...)
# Add test cases
ProblemTestCase.objects.create(dataset=prob, order=1,
    input_file='1.in', output_file='1.out', type='C', points=100)
```

## Contests

Create contests through the admin interface at `/admin/judge/contest/` or via script:

```python
from judge.models import Contest, ContestProblem
c = Contest.objects.create(key='mycontest', name='My Contest', ...)
ContestProblem.objects.create(contest=c, problem=prob, order=1, points=100)
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| 502 Bad Gateway | Run `docker compose restart site` |
| PDF 404 | Check nginx config: `location /pdf/ { alias /media/; }` |
| Judge offline | Verify `vnoj_judge` container is running and can reach `bridged:9999` |
| Migration errors | Ensure `lxml_html_clean` is installed: `pip install lxml_html_clean` |
| Tablespace error | Use Docker volume (not bind mount) for MySQL data |
| Static files missing | Run `collectstatic --noinput` after changes |

## License

Based on [DMOJ](https://github.com/DMOJ/online-judge) - see original LICENSE.
