# sge plan: review a migration before it runs

`sge plan` reads the database and your scripts the way `migrate` would, changes nothing, and writes what `migrate` would change as Markdown: per table, the scripts that change it - DDL with its columns, DML with the rows it changes, counted in the database as it is now. Put it in front of a reviewer, and run `migrate` once they approve.

`Schema Guard Enterprise (sge) 2.3 · PostgreSQL · MIT`

## Run it

```bash
sge -c db/prod.cfg plan > plan.md
```

The plan goes to standard output, the log to standard error. It takes every option `migrate` takes - the connection, the source of the scripts (a directory, a git repository, a zip archive), the history table - and two of its own:

| option | what it does |
|---|---|
| `--jira_prefix=PROJ,OPS` | groups the plan by the Jira tickets named in the scripts' comments (`-- PROJ-123`), in the schema files' comments for their data files too |
| `--dry_run_to=plan.sql` | writes the SQL `migrate` would run to a file as well - a script as `\i` of its file |

`--createdb` is not followed: a plan creates no database. A database user that can read the tables and the catalog is enough - every question the plan asks is a `SELECT` in a read-only transaction.

With the Docker image, on a database the machine reaches:

```bash
docker run --rm --network host -e DB_PASSWORD -v "$PWD:/work" dbinvent/sge:2.3.0 -c db/prod.cfg plan > plan.md
```

`sge plan` exits with an error, and writes no plan, when `migrate`'s dry run fails: the database cannot be reached, a script cannot be read, a schema file is not valid.

## What it says

```markdown
## sge plan: app@db.example.com
tables: 4, scripts: 3

dml: rows counted in the database as it is now; N statements: their rows not counted - on a table the plan creates, or a statement that cannot be counted

### PROJ-7

- **public.audit**
  - `V1_4__orders.sql`: ddl create (id, at) | dml update 1 statement
- **public.orders**
  - `V1_4__orders.sql`: ddl alter: add email | dml insert 1, update 4, delete 1
- **public.orders_tmp**
  - `V1_4__orders.sql`: dml truncate 120

### No ticket

- **public.customers**
  - `D1__customers.csv`: dml insert 2, update 1, delete 1
- **other**
  - `V1_4__orders.sql`: ddl create extension pgcrypto
```

- **The title** names the database and its host; the line under it counts the tables changed and the scripts that change them.
- **By ticket**, with `--jira_prefix`: each ticket the scripts' comments name, then the scripts that name none under *No ticket*. A script that names two tickets is under both. Without `--jira_prefix`, the tables alone.
- **By table**: `schema.table`, and under it each script that changes it - its file, with its version when the file name does not carry it.
  - `ddl` - what changes in its definition: `create (columns)`, `alter: add email, drop old`, an index, a trigger, a policy, a grant, a sequence set.
  - `dml` - the rows: `insert`, `update`, `delete`, and `truncate` - every row the table has. A count that is 0 is not shown. Where the rows cannot be counted (below), the statements are: `update 2 statements`; `update 4 + 1 statement` is four rows counted, and one statement whose rows are not.
- **other** - what belongs to no table: a schema, an extension, a function, a type, an index whose table the run does not tell - and the statements the plan could not read (`statements not read: N`), and a script it could not read at all.
- `after_each.sql ×3` - a script that runs after each version, with the times it runs.
- **sge's history table** - created the first time `migrate` runs against a database.

## How it decides

**Schema files** (`S1__schema.yaml`): Schema Guard compares each table with the database and says what it would create or alter - the same decision `migrate` makes, not SQL read back.

**Data files** (`D1__customers.csv`): each row by its primary key. A key the table has is no insert - it is an update when the file's rule updates (`--etl=clean` or `force`, the table named in `--snapshot_data`); a key it has not is an insert; `clean` deletes the rows whose key is not in the file, and those are counted. A table with no primary key: a row the table has already, every column equal, is no insert. A row the file has twice is one. A table the plan creates takes every row as new.

**Scripts** (`V1_2__x.sql`, `R__y.sql`, `P1__z.sql`, after scripts) are read statement by statement:

- **DDL on a condition** - `CREATE ... IF NOT EXISTS`, `DROP ... IF EXISTS`, `ALTER TABLE IF EXISTS`, `ADD COLUMN IF NOT EXISTS`, `DROP COLUMN IF EXISTS`, `DROP CONSTRAINT IF EXISTS`, `DROP TRIGGER`, `POLICY`, `FUNCTION`, `TYPE`, `SCHEMA`, `EXTENSION ... IF [NOT] EXISTS` - is asked of the database, and one whose condition does not hold is no change and is not shown. What the run did before counts first: a table an earlier script creates makes a later `CREATE TABLE IF NOT EXISTS` of it no change; a table it drops and creates again has none of the old one's columns or indexes; a table it renames keeps them under its new name.
- **A DO block** is read for its statements: each `EXECUTE '...'`, else the statement itself. Its `IF ... THEN ... ELSE ... END IF` is asked of the database, and only the branch the condition takes is shown - `NULL` takes `ELSE`, as plpgsql does.
- **UPDATE and DELETE** are counted with `SELECT count(*)` and their own `WHERE` - with a `FROM`, a `USING`, a `WITH` - on the table they change.
- **TRUNCATE** is counted as every row its table has - each table it names, `ONLY` as written.
- **INSERT ... VALUES** inserts its rows. With `ON CONFLICT (columns)`, a row whose key the table has is no insert - `DO UPDATE` updates it; each key is counted once.
- **INSERT ... SELECT with no table**, the seed pattern - `INSERT INTO t SELECT 1, 'a' WHERE NOT EXISTS (...)` - is counted the same way: as many rows as its `WHERE` lets through.
- **SET search_path** is followed: a name written without its schema is found the way Postgres finds it, and shown in the schema it is in.

**Safe to run against production.** Each question - is it there, how many rows, which branch - runs in a read-only transaction of its own, rolled back: nothing a condition or a `WHERE` would write gets through, not a `nextval()`, not a setting. Its locks go with it. A question waits for a lock 10 seconds at most and runs 60 at most; one the database refuses, or that takes longer, costs no other - its statement is counted instead of its rows.

## Approval in GitHub Actions

A reusable workflow cannot hold an approval step for the workflow that calls it, so this one is a sample to copy into your repository, as `.github/workflows/db.yml`. It runs in three jobs - **plan, approval with a double check, migrate** - with the `dbinvent/sge` image alone:

1. **plan** runs `sge plan` and puts the Markdown on the run's summary page; the plan and its SQL are kept as the run's artifact `plan`.
2. **approval** waits for the required reviewers of the `production` environment. A plan with nothing to change asks for none.
3. **migrate** runs once they approve - the double check first: it makes the plan again, and stops with "the database changed since the plan" if it differs from the one approved; else it migrates. The double check is on unless a run turns it off.

Set up two environments first, in the repository's **Settings → Environments**:

| environment | holds | set |
|---|---|---|
| `db` | the database's password, secret `DB_PASSWORD` | **Deployment branches**: `main` alone |
| `production` | no secrets | **Required reviewers**: the people who approve a migration |

The password is in an environment rather than the repository: a repository secret is any branch's. `production` holds no secret, so a run asks for approval once.

```yaml
# .github/workflows/db.yml - plan, approval with a double check, migrate
name: db
on:
  push:
    branches: [main]
    paths: ["db/**"]
  workflow_dispatch:                    # by hand, with its inputs
    inputs:
      target:
        description: "version to migrate to - empty: all"
        default: ""
      double_check:
        description: "plan again before migrate, stop if the database changed"
        type: boolean
        default: true
permissions:
  contents: read
env:
  TARGET: ${{ inputs.target }}          # sge --target: scripts after it wait
  DOUBLE_CHECK: ${{ github.event_name != 'workflow_dispatch' || inputs.double_check }}
jobs:
  plan:                                 # what migrate would change, on the run's summary page
    runs-on: ubuntu-latest
    environment: db                     # the database's password, main alone
    outputs:
      changes: ${{ steps.plan.outputs.changes }}
    steps:
      - uses: actions/checkout@v7
        with: {persist-credentials: false}
      - id: plan
        env:
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
        run: |
          : > plan.sql
          docker run --rm --network host --user "$(id -u):$(id -g)" -e HOME=/tmp -e DB_PASSWORD \
            -v "$PWD:$PWD" -w "$PWD" dbinvent/sge:2.3.0 -c db/prod.cfg ${TARGET:+--target="$TARGET"} \
            --dry_run_to=plan.sql --jira_prefix=PROJ plan > plan.md
          cat plan.md >> "$GITHUB_STEP_SUMMARY"
          grep -q '[^[:space:]]' plan.sql && echo changes=yes >> "$GITHUB_OUTPUT" || true
      - uses: actions/upload-artifact@v7
        with: {name: plan, path: "plan.*"}

  approval:                             # the reviewers read the plan and approve it
    needs: plan
    if: needs.plan.outputs.changes == 'yes'
    runs-on: ubuntu-latest
    environment: production             # required reviewers, no secrets
    steps:
      - run: echo "migration approved"

  migrate:                              # the double check, then migrate
    needs: approval
    runs-on: ubuntu-latest
    environment: db
    concurrency: {group: db, cancel-in-progress: false}
    steps:
      - uses: actions/checkout@v7
        with: {persist-credentials: false}
      - uses: actions/download-artifact@v8
        if: env.DOUBLE_CHECK == 'true'
        with: {name: plan, path: approved}
      - env:
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
        run: |
          sge() { docker run --rm --network host --user "$(id -u):$(id -g)" -e HOME=/tmp -e DB_PASSWORD \
            -v "$PWD:$PWD" -w "$PWD" dbinvent/sge:2.3.0 -c db/prod.cfg ${TARGET:+--target="$TARGET"} "$@"; }
          if [ "$DOUBLE_CHECK" = true ]; then   # a new plan, the one approved
            : > now.sql
            sge --dry_run_to=now.sql plan > /dev/null
            cmp -s now.sql approved/plan.sql || { echo "::error::the database changed since the plan"; exit 1; }
          fi
          sge migrate
```

Run by hand - **Actions → db → Run workflow** - it takes two inputs, both optional; a push runs with both as they default:

| input | what it does | default |
|---|---|---|
| `target` | the version to migrate to: `sge --target`, in the plan and the migrate alike - the scripts after it wait for a later run | empty: all of them |
| `double_check` | the plan made again before migrate, compared with the one approved | on |

A run by hand from a branch other than `main` stops at its first job: the `db` environment deploys `main` alone.

Fit it to your repository:

- `db/**` and `db/prod.cfg` - where your scripts and their configuration are. The configuration holds the connection, `key=value` a line, the password aside: sge reads it from `DB_PASSWORD`.
- `--network host` - the runner reaches the database as the host does. On a self-hosted runner in the database's network, nothing more is needed; on GitHub's runners the database must be reachable from them.
- `--jira_prefix=PROJ` - your Jira project keys, or drop it for a plan by table.
- `dbinvent/sge:2.3.0` - pin the version you tested; `2.3` follows its fixes.
- `git` is not in the image: for a `source_git` script source, build an image `FROM dbinvent/sge` with `apk add git openssh-client`.

**Reviewing.** The run stops at **approval**, waiting. A reviewer opens the run, reads the plan on its summary page - by ticket, by table, the rows each script changes - and approves it under **Review deployments**, or rejects it. The plan stays with the run as its artifact, with the SQL `migrate` would run.

**The double check.** Between approval and migrate the database can change - another run, a script by hand, an application. **migrate** makes the plan again and compares its SQL with the approved one: a difference stops the run before anything is applied - "the database changed since the plan" - and a new run makes a new plan to approve. Turned off (`double_check` unchecked), migrate applies what the database is then, approved or not: for a database nothing else changes. `concurrency` lets one migrate run at a time.

## What is not counted as DML

A plan reads scripts; it does not run them. What it cannot read, it does not count - and says so where it can: the statements whose rows it cannot count, as `N statements`; those it cannot read at all, as `statements not read`. Read the plan with these in mind:

- **COPY is not read.** `COPY ... FROM STDIN` and `COPY ... FROM 'file'` - their rows are not in the plan, and the table is not shown for them. Their data in a script is skipped, not taken for SQL.
- **INSERT ... SELECT from a table is not read.** Its rows are known only by running it. An `INSERT ... SELECT` of no table - values, with or without a `WHERE` - is counted.
- **psql's commands are not read**: `\copy`, `\set`, `\gexec`, `\o` and the rest. `\i` and `\ir` are followed, and the SQL of the files they include is read.
- **A restore is not seen**: `pg_restore`, a dump loaded by `psql` outside a script, data changed by hand or by another application.
- **Rows on a table the plan creates are not counted** - its statements are, `update 1 statement`: the database has not got the table yet. The same for a table the plan renames, or drops and creates again. A data file's rows on such a table are all counted as inserts.
- **Counts are against the database as it is now.** One script's rows are not added for the next: an `INSERT` in one script and an `UPDATE` of those rows in the next counts only the rows there now. Two statements that change the same row are two.
- **What runs inside the database is not seen**: rows a trigger changes, rows an `ON DELETE CASCADE` deletes, what a function a script calls changes, a rule, a policy that hides rows from the plan's user.
- **Dynamic SQL is not read**: `EXECUTE format(...)` in a DO block, a function's body. Such a statement is counted under `statements not read`.
- **MERGE's rows are not counted** - its statements are, `merge 1 statement`.
- **Statements, not rows, by the statement's form:**
  - an `UPDATE` or a `DELETE` in a `WITH` that writes, with a join on its table, with `LIMIT` or `ORDER BY`;
  - `INSERT ... ON CONFLICT ON CONSTRAINT`, or with no columns named, or a key that is not a literal;
  - `ON CONFLICT DO UPDATE ... WHERE` - its updates;
  - a count the database refuses, waits more than 10 seconds for a lock for, or takes more than 60 seconds over.
- **An update is a row its `WHERE` selects**, as Postgres counts it - a row set to the values it has is updated too.
- **TRUNCATE ... CASCADE** - the rows of the tables it names are counted; those of the tables it truncates through their foreign keys are not, nor are those tables shown.
- **A schema file's inline data** (`data:` in the YAML) - its rows as written, as inserts; they are not looked up in the table.
- **A DO block's IF**: a condition the database cannot answer on its own - one with a plpgsql variable - takes both branches; an `IF` with `ELSIF` is read as if every branch ran.
- **The same DDL twice** on one table, in one script, is shown once.

**Between the plan and the migrate**, the database can change, and the migrate does what the database is then. The workflow's double check makes the plan again before it migrates, and stops if it differs from the one approved - unless a run turns it off.
