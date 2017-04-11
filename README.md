![image](https://nazavode.github.io/img/lifeboat-banner.png)

TracBoat
========

A life boat to escape the Trac ocean liner.

**TracBoat** is a toolbox for exporting whole Trac instances, saving
them and migrating them to other platforms.

Features
--------

-   Export Trac projects along with issues, issue changelogs (with
    attachments) and wiki pages (with attachments)
-   Create GitLab users, projects and namespaces involved in the
    migration process
-   Export Trac projects to local files (in json, toml, Python literal
    or Python pickle formats)
-   Migrate Trac projects directly from remote instances as well as from
    previously created export files
-   Migrate Trac projects to a live GitLab instance
-   Migrate Trac projects to a mocked GitLab on the file system to check
    for correctness

Installation
------------

If you want to **install from source**, doing it in a `virtualenv` is
*highly recommended*. After having this repo cloned, just type:

```
$ cd tracboat
$ virtualenv -p python2.7 VENV
$ source VENV/bin/activate
$ pip install -r requirements.txt
$ pip install -e .
```

Dependencies
------------

-   Python &gt;= 2.7 or Python 3.x
-   [peewee](https://pypi.python.org/pypi/peewee)
-   [psycopg2](https://pypi.python.org/pypi/psycopg2)
-   [six](https://pypi.python.org/pypi/six)
-   [click](https://pypi.python.org/pypi/click)
-   [toml](https://pypi.python.org/pypi/toml)
-   [pymongo](https://pypi.python.org/pypi/pymongo)

Getting started
---------------

Every command line option can be specified as an environment variable,
so the following command:

```
$ tracboat users --trac-uri=https://mytrac/xmlrpc --no-ssl-verify
```

...is exactly the same as the following:

```
$ export TRACBOAT_TRAC_URI=https://mytrac/xmlrpc
$ export TRACBOAT_SSL_VERIFY=0
$ tracboat users
```

Another way to conveniently configure `tracboat` is with a configuration
file in [TOML](https://github.com/toml-lang/toml) format. Providing a
file

```
$ cat mytrac.toml
[tracboat]
trac_uri = "https://mytrac/xmlrpc"
ssl_verify = false
$ tracboat --config-file=mytrac.toml users
```

Please note that when a value is specified more that once, the priority
order considered is the following:

1.  command line option;
2.  environment variable;
3.  configuration file;
4.  default value.

If you are *very* curious about how to play with command line options,
have a look to the [click documentation](http://click.pocoo.org/).

### Collecting Trac users

```
$ tracboat users --trac-uri=http://localhost/xmlrpc
```

### Export a Trac instance

```
$ tracboat export --trac-uri=http://localhost/xmlrpc --format=json --out-file=myproject.json
```

### Migrate to GitLab

```
$ cat awesomemigration.toml
[tracboat]
from-export-file = "myexportedtracproject.json"
gitlab-project-name = "migrated/myproject"
gitlab-version = "9.0.0"
gitlab_db_password = "Բարեւ աշխարհ"
$ tracboat --config-file=awesomemigration.toml migrate
```

### Migrating users

During a migration we need to map Trac usernames to GitLab user accounts
to keep all associations between issues, changelog entries and wiki
pages and their authors. By default, all Trac usernames are mapped to a
single GitLab user, the so called *fallback user*. This way you'll end
up with a migrated project where all activity looks like it come from a
single user. Not so fancy, but definitely handy if you just care about
content. You can specify a custom fallback username with the proper
option:

```
$ cat config.toml
[tracboat]
fallback_user = "bot@migration.gov"
```

As usual, the same behaviour can be obtained via command line option or
environment variable. So doing this:

```
$ export TRACBOAT_FALLBACK_USER=bot@migration.gov
$ tracboat migrate
```

...is the same as doing this:

```
$ tracboat migrate --fallback-user=bot@migration.gov
```

#### Mapping users

When you want your Trac users mapped to a GitLab user (and to the
corresponding account) you need to specify a custom *user mapping*, or
an association between a Trac username and a GitLab account. You can use
a key-value section in the configuration file:

```
$ cat config.toml
[tracboat.usermap]
    tracuser1 = "gitlabuser1@foo.com"
    tracuser2 = "gitlabuser2@foo.com"
    tracuser3 = "gitlabuser1@foo.com"
```

In this case, every action that in the Trac project belongs to
`tracuser1`, in the migrated GitLab project will end up as being
authored by `gitlabuser1@foo.com`.

You can add extra mappings using the `--umap` command line option, so
doing like this:

```
$ tracboat migrate --umap tracuser1 gitlabuser1@foo.com --umap tracuser2 gitlabuser2@foo.com ...
```

...obtains exactly the same behaviour as with the configuration file
above. *Remember that for repeated values, command line takes precedence
over configuration file.*

#### Custom user attributes

If a user doesn't exist in GitLab yet, he will be created during the
migration process. However, creating a new GitLab account is a fairly
complex affair: you can specify social accounts, biography, links and [a
lot of other
stuff](https://docs.gitlab.com/ce/api/users.html#user-creation). If you
don't say anything about how an user should be created, Tracboat uses
some defaults. However you can throw a proper section in the
configuration file to tweak those user creation attributes:

```
$ cat config.toml
[tracboat.users.default]
    admin = false
    external = true
    website_url = "http://www.foo.gov"
```

Those values will be applied to *all* new accounts created during the
migration process. However, you can spceify additional `user`
subsections to precisely control which values would be used for a
particular account:

```
$ cat config.toml
[tracboat.users.default]
    admin = false
    external = true
    website_url = "http://www.foo.gov"

[tracboat.users."theboss@foo.gov"]
    username = "theboss"
    bio = "Hi. I am the boss here."
    admin = true
    twitter = "@theboss"
    external = false
```

In this case, all users are going to be created with the attributes
contained in the `[tracboat.users.default]` section except fot the boss
that asked explicitly for some extra goodies.

Example
-------

This is a fairly complete configuration example with a usermap and
custom user attributes. You can find additional examples in the
`examples/` directory.

```
# Tracboat will look for values in the [tracboat] section only, so
# you can merge in a single file values for other applications.

[tracboat]

# The Trac instance to be crawled.
# If you have any secrets in the URL (just like in this case,
# our password is in plain text), consider using the corresponding
# environment variable TRACBOAT_TRAC_URI to avoid having secrets in
# the configuration file.
trac_uri = "https://myuser:MYPASSWORD@localhost/login/xmlrpc"

# Disable ssl certificate verification (e.g.: needed with self signed certs).
ssl_verify = false

# The GitLab project name.
# Can be specified as a path, subdirectories will be treated as GitLab
# namespaces.
gitlab_project_name = "migrated/myproject"

# The fallback user, used when a Trac username has no entry in the
# [tracboat.usermap] section.
fallback_user = "bot@tracboat.gov"

# Users configuration.
# Every section beyond this point can be passed in separate TOML files
# with repeated --umap-file command line options or directly here:
#
# umap_file = ['users1.toml', 'users2.toml']

# The Trac -> GitLab user conversion mapping.
# It is *highly* recommended to use a valid email address for the GitLab part
# since by default each account will be created with a random password
# (you need a valid address for the password reset procedure to work properly).
[tracboat.usermap]
    tracuser1 = "gitlabuser1@foo.com"
    tracuser2 = "gitlabuser2@foo.com"
    tracuser3 = "gitlabuser3@foo.com"
    tracuser4 = "gitlabuser4@foo.com"

[tracboat.users]
# GitLab users attributes.
# This section allows to specify custom attributes
# to be used during GitLab user creation. Accepted values are
# listed here:
# https://docs.gitlab.com/ce/api/users.html#user-creation

[tracboat.users.default]
    # This 'default' section specifies attributes applied
    # to all new GitLab users.
    external = true

[tracboat.users."gitlabuser4@foo.com"]
    # This section affects a specific user (in this case "gitlabuser4@foo.com").
    # These key-value entries will be merged with those in the
    # [tracboat.users.default] section. For repeated values, those specified
    # here will prevail.
    #
    # There are some mandatory values that must be specified
    # for each user, otherwise the following default values
    # will be used:
    #
    # username = ...
    #     Defaults to the user part of the GitLab email address
    #     (e.g. "gitlabuser4" for "gitlabuser4@foo.com").
    #
    # encrypted_password = ...
    #     Defaults to a random password (at the first login the user must carry
    #     out a password reset procedure). Anyway, you are *highly* discouraged
    #     to specify secrets here, please stick to the default behaviour.
    username = "theboss"
    bio = "Hi. I am the boss here."
    admin = true
    twitter = "@theboss"
    external = false  # this value overrides tracboat.users.default.external

[tracboat.users."bot@tracboat.gov"]
    # This section affects the fallback user, used when a Trac
    # username has no entry in the [tracboat.usermap] section.
    username = "migration-bot"
    bio = "Hi. I am the robot that migrated all your stuff."
    admin = true
    external = false
```

Credits
-------

The initial inspiration and core migration logic comes from the
[trac-to-gitlab](https://github.com/moimael/trac-to-gitlab) project by
[Maël Lavault](https://github.com/moimael): this project was born from
heavy cleanup and refactoring of that original code, so this is why this
spinoff inherited its
[GPLv3](https://www.gnu.org/licenses/gpl-3.0.en.html) license.

Changes
-------

### 0.2.0-alpha *(unreleased)*

#### Added

-   Project import
-   Migration to mock GitLab on file system
-   Creation of missing GitLab users, namespaces and projects
-   Custom user attributes (#17)
