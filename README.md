# N8N Workflow Multiple Ports

![Last Commit](https://img.shields.io/github/last-commit/Siphon880gh/multiple-n8n/main)
<a target="_blank" href="https://github.com/Siphon880gh" rel="nofollow"><img src="https://img.shields.io/badge/GitHub--blue?style=social&logo=GitHub" alt="Github" data-canonical-src="https://img.shields.io/badge/GitHub--blue?style=social&logo=GitHub" style="max-width:8.5ch;"></a>
<a target="_blank" href="https://www.linkedin.com/in/weng-fung/" rel="nofollow"><img src="https://img.shields.io/badge/LinkedIn-blue?style=flat&logo=linkedin&labelColor=blue" alt="Linked-In" data-canonical-src="https://img.shields.io/badge/LinkedIn-blue?style=flat&amp;logo=linkedin&amp;labelColor=blue" style="max-width:10ch;"></a>
<a target="_blank" href="https://www.youtube.com/@WayneTeachesCode/" rel="nofollow"><img src="https://img.shields.io/badge/Youtube-red?style=flat&logo=youtube&labelColor=red" alt="Youtube" data-canonical-src="https://img.shields.io/badge/Youtube-red?style=flat&amp;logo=youtube&amp;labelColor=red" style="max-width:10ch;"></a>

By Weng Fei Fung (Weng).

This guide explains running multiple n8n instances at different ports sharing the same database. Our focus is running locally, but you can reverse engineer this guide to run multiple ports online..

Because we are focusing locally, we will learn how to clone your Heroku docker n8n locally. We also cover how to dump the Heroku's Postgres into the localhost Postgres (that's installed via a Postgres image at compose.yml). 

Remember that the database is a clone of Heroku's n8n database. So for running locally, it won't affect your online version.

Prerequisite: Have a working workflow on Heroku. That Heroku instance is a dockerized n8n.

Use cases:
- Taking a working Heroku n8n workflow offline to use it as an internal tool privately.
- Using the workflow across multiple ports without it being online in order to minimize costs such as extra Heroku dynos or scale up cloud costs.

## Prerequisites

- Docker and Docker Compose installed
- Heroku CLI installed (`brew install heroku` on macOS)
- Access to your Heroku app (`n8n-tc`)
- Authenticated with Heroku container registry (see Quick Start step 0)

## Architecture

This setup runs:
- PostgreSQL database on port 5433 (mapped from container's 5432 to avoid conflicts)
- n8n5002 instance on http://localhost:5002
- n8n5003 instance on http://localhost:5003

Both n8n instances share the same PostgreSQL database (like on Heroku).

**Note:** PostgreSQL uses port 5433 locally to avoid conflicts with any existing PostgreSQL installation on your Mac.

## Quick Start

1. **Authenticate with Heroku (first time only):**
   ```bash
   heroku login
   heroku container:login
   ```

2. **Setup Compose:**

Create a `compose.yml` based on `compose.sample.yml`, filling in your app name and other details. Note the compose.sample.yml is geared towards Macbook Pro 2021 and hence it's linux/amd64 platformized. Usually you DO NOT need to adjust the username and password - Docker will create such credentials if doesn't exist.

3. **Start the services:**
   ```bash
   docker compose up -d
   ```

4. **Check the logs:**
   ```bash
   docker compose logs -f n8n5002
   docker compose logs -f postgres
   ```

5. **Access n8n:**
   - Instance 1: http://localhost:5002
   - Instance 2: http://localhost:5003

Note it may say "Site unable to reach" for the first few minutes depending on your computer's speed. You may run daemonless to get a feel when n8n finishes initializing.

6. **Verify n8n is running:**
   ```bash
   curl -I http://localhost:5002
   # Should return HTTP/1.1 200 OK
   ```

7. **Stop the services:**
   ```bash
   docker compose down
   ```

## Database Migration from Heroku

### Option 1: Using Heroku pg:pull (Recommended)

This is the easiest method and works best for most cases.

1. **Log in to Heroku:**
   ```bash
   heroku login
   ```

2. **Start your local PostgreSQL:**
   ```bash
   docker compose up -d postgres
   ```

3. **Pull the database from Heroku:**
   ```bash
   heroku pg:pull DATABASE_URL postgresql://n8n:n8n_password@localhost:5432/n8n -a n8n-tc
   ```

   This command:
   - DATABASE_URL: Do NOT change this part of the command unless it's a different variable name at Heroku. Connects to your Heroku database via the connection string in DATABASE_URL.
 	  - Your n8n docker setup at Heroku should have installed resource service add-on called Heroku Postgres. That would have added the config variable DATABASE_URL. 
   - Downloads all data
   - Imports it into your local PostgreSQL

   Errors
   - If authentication fails, pay attention to the n8n user that's in docker.yml. Try the password again in the connection string.
   - If says n8n already exists, you'll need to delete that table from postgresql. Log into your postgresql server via PgAdmin. Register a server so it becomes available in the Object Explorer navigation panel on the left. Name it whatever you want, but under Connection tab that "Host name/address" can be local host. Fill in Port, Username, and Password. Then you can view the n8n table under Schemas -> Tables -> n8n. You right click and delete it. Then you should be able to pull and dump into the local n8n table with the above command.

4. **Start n8n instances:**
   ```bash
   docker compose up -d
   ```

### Option 2: Using pg_dump and pg_restore

This method gives you more control and creates a backup file.

1. **Create a backup on Heroku:**
   ```bash
   heroku pg:backups:capture -a n8n-tc
   heroku pg:backups:download -a n8n-tc
   ```

   This downloads a file named `latest.dump`

2. **Start your local PostgreSQL:**
   ```bash
   docker compose up -d postgres
   ```

3. **Restore the backup to local database:**
   ```bash
   # Wait a few seconds for PostgreSQL to be ready
   sleep 5
   
   # Restore the dump
   docker compose exec -T postgres pg_restore -U n8n -d n8n --clean --if-exists --no-owner --no-acl < latest.dump
   ```

   Or alternatively:
   ```bash
   pg_restore -h localhost -p 5433 -U n8n -d n8n --clean --if-exists --no-owner --no-acl latest.dump
   ```
   (Password: `n8n_password`)

4. **Start n8n instances:**
   ```bash
   docker compose up -d
   ```

### Option 3: Manual SQL Export/Import

If the above methods don't work:

1. **Export from Heroku (creates SQL file):**
   ```bash
   heroku pg:backups:capture -a n8n-tc
   heroku pg:backups:download -a n8n-tc -o heroku_backup.dump
   
   # Convert to SQL format
   pg_restore --no-owner --no-acl -f heroku_backup.sql heroku_backup.dump
   ```

2. **Start local PostgreSQL:**
   ```bash
   docker compose up -d postgres
   sleep 5
   ```

3. **Import the SQL file:**
   ```bash
   docker compose exec -T postgres psql -U n8n -d n8n < heroku_backup.sql
   ```

## Verifying the Migration

1. **Check database connection:**
   ```bash
   docker compose exec postgres psql -U n8n -d n8n -c "\dt"
   ```

   You should see n8n tables listed.

2. **Check record counts:**
   ```bash
   docker compose exec postgres psql -U n8n -d n8n -c "SELECT COUNT(*) FROM workflow_entity;"
   docker compose exec postgres psql -U n8n -d n8n -c "SELECT COUNT(*) FROM execution_entity;"
   ```

3. **Access n8n UI:**
   - Visit http://localhost:5002
   - Your workflows should be visible
   - Check credentials are working

## Troubleshooting

### Cannot pull Heroku image / Authentication errors

If you see errors pulling the image, authenticate with Heroku:
```bash
heroku login
heroku container:login
docker compose pull
docker compose up -d
```

### n8n won't start / Database connection refused

Check PostgreSQL is running and healthy:
```bash
docker compose ps
docker compose logs postgres
```

Wait for the message: `database system is ready to accept connections`

### Permission denied errors during restore

Add `--no-owner --no-acl` flags to pg_restore commands.

### Schema version mismatch

If Heroku is running a newer/older version of n8n:

1. Check Heroku n8n version:
   ```bash
   heroku run -a n8n-tc "n8n --version"
   ```

2. Update your Docker image to match (edit `compose.yml` if needed)

### Workflows not appearing

Check the data was imported:
```bash
docker compose exec postgres psql -U n8n -d n8n
```

Then run:
```sql
SELECT id, name, active FROM workflow_entity;
```

### Access denied / Authentication errors

Make sure credentials in `compose.yml` match what you're using:
- Database: `n8n`
- User: `n8n`
- Password: `n8n_password`

## Syncing Data Back to Heroku

To push local changes back to Heroku:

```bash
heroku pg:push postgresql://n8n:n8n_password@localhost:5433/n8n DATABASE_URL -a n8n-tc
```

**⚠️ WARNING:** This will overwrite your Heroku database!

## Database Management

### Backup local database:
```bash
docker compose exec postgres pg_dump -U n8n n8n > backup_$(date +%Y%m%d_%H%M%S).sql
```

### Restore local backup:
```bash
docker compose exec -T postgres psql -U n8n -d n8n < backup_20250108_120000.sql
```

### Reset local database (fresh start):
```bash
docker compose down
docker volume rm multi_postgres_data
docker compose up -d
```

### Access PostgreSQL directly:
```bash
docker compose exec postgres psql -U n8n -d n8n
```

## Environment Variables

Key environment variables in `compose.yml`:

### PostgreSQL:
- `POSTGRES_DB=n8n` - Database name
- `POSTGRES_USER=n8n` - Database user
- `POSTGRES_PASSWORD=n8n_password` - Database password

### n8n:
- `DATABASE_URL=postgresql://n8n:n8n_password@postgres:5432/n8n` - **Main database connection string** (used by Heroku image)
- `DB_TYPE=postgresdb` - Use PostgreSQL
- `DB_POSTGRESDB_HOST=postgres` - Connect to postgres service (within Docker network)
- `DB_POSTGRESDB_DATABASE=n8n` - Database name
- `N8N_URL` - Public URL for webhooks and UI
- `PORT=5678` - Internal port n8n listens on

**Important:** The Heroku n8n image requires `DATABASE_URL` to be set. The port in DATABASE_URL is `5432` (internal Docker network), not `5433` (external host port).

## Notes

- Both n8n instances (5002 and 5003) share the same database
- Data persists in Docker volumes even after `docker compose down`
- To fully reset: `docker compose down -v` (removes all data)
- PostgreSQL data is stored in the `postgres_data` volume
- n8n config/keys are stored in `n8n_data_5002` and `n8n_data_5003` volumes

