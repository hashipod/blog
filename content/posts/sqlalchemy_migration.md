---
author: jieteki
categories:
- 技术文章
date: 2016-12-17T13:10:48+08:00
description: ""
keywords:
- xxx
title: Sqlalchemy migration
url: ""
---

Basic usage of sqlalchemy migration usage.

<!--more-->


1\. create repository and versioned db.

```sh
➜
➜ migrate create my_repo "Example project"
➜ migrate manage manage.py --repository=my_repo --url=sqlite:///my_repo.db
➜ ll
total 8
drwxr-xr-x  26 chenjie  staff   884B 11  4 10:57 cinder
-rw-r--r--   1 chenjie  staff   166B 11  4 14:11 manage.py
drwxr-xr-x   8 chenjie  staff   272B 11  4 14:11 my_repo
➜ python manage.py version_control
➜ ll
total 32
drwxr-xr-x  26 chenjie  staff   884B 11  4 10:57 cinder
-rw-r--r--   1 chenjie  staff   166B 11  4 14:11 manage.py
drwxr-xr-x   8 chenjie  staff   272B 11  4 14:11 my_repo
-rw-r--r--   1 chenjie  staff    12K 11  4 14:12 my_repo.db
➜ sqlite3 my_repo.db
SQLite version 3.14.0 2016-07-26 15:17:14
Enter ".help" for usage hints.
sqlite> .tables
migrate_version
sqlite> .schema migrate_version
CREATE TABLE migrate_version (
    repository_id VARCHAR(250) NOT NULL,
    repository_path TEXT,
    version INTEGER,
    PRIMARY KEY (repository_id)
);
sqlite> ^D
➜
```

2\. script and migrate

```python
➜ python manage.py script "add account table"
➜ python manage.py test
Upgrading...
done
Downgrading...
done
Success
➜ cat my_repo/versions/001_add_account_table.py
from sqlalchemy import Table, Column, Integer, String, MetaData

meta = MetaData()
account = Table(
    'account', meta,
    Column('id', Integer, primary_key=True),
    Column('login', String(40)),
    Column('passwd', String(40)),
)


def upgrade(migrate_engine):
    # Upgrade operations go here. Don't create your own engine; bind
    # migrate_engine to your metadata
    meta.bind = migrate_engine
    account.create()

def downgrade(migrate_engine):
    # Operations to reverse the above upgrade go here.
    meta.bind = migrate_engine
    account.drop()

➜ python manage.py version
1
➜ python manage.py db_version
0
➜ python manage.py upgrade
0 -> 1...
done
➜ python manage.py db_version
1
➜

```

3\. modify the table

```python
➜ python manage.py script "add email column"
➜ cat my_repo/versions/002_add_email_column.py
from sqlalchemy import Table, MetaData, String, Column


def upgrade(migrate_engine):
    meta = MetaData(bind=migrate_engine)
    account = Table('account', meta, autoload=True)
    emailc = Column('email', String(128))
    emailc.create(account)


def downgrade(migrate_engine):
    meta = MetaData(bind=migrate_engine)
    account = Table('account', meta, autoload=True)
    account.c.email.drop()
➜ python manage.py test
Upgrading...
done
Downgrading...
done
Success
➜ python manage.py upgrade
1 -> 2...
done
➜ python manage.py db_version
2
➜ python manage.py version
2
➜

```

4\. Writings .sql scripts

HAVE ERROR. TODO.

```sh

➜ python manage.py script_sql mysql add_age
➜ cat my_repo/versions/003_add_age_mysql_upgrade.sql

ALTER TABLE account
ADD age int
➜ cat my_repo/versions/003_add_age_mysql_downgrade.sql

ALTER TABLE account
DROP COLUMN age
➜ python manage.py test
Upgrading...
Traceback (most recent call last):
  File "manage.py", line 5, in <module>
    main(url='sqlite:///my_repo.db', debug='False', repository='my_repo')
  File "/usr/local/lib/python2.7/site-packages/migrate/versioning/shell.py", line 209, in main
    ret = command_func(**kwargs)
  File "<decorator-gen-7>", line 2, in test
  File "/usr/local/lib/python2.7/site-packages/migrate/versioning/util/__init__.py", line 160, in with_engine
    return f(*a, **kw)
  File "/usr/local/lib/python2.7/site-packages/migrate/versioning/api.py", line 218, in test
    script = repos.version(None).script(engine.name, 'upgrade')
  File "/usr/local/lib/python2.7/site-packages/migrate/versioning/version.py", line 207, in script
    "There is no script for %d version" % self.version
AssertionError: There is no script for 3 version
```
