= PostgreSQL with client certificates

== Info

=== Server

SSL have to be set up ([[HowToPostgreSQLoverSSL]]).

Most things described in
http://www.postgresql.org/docs/current/static/runtime-config-connection.html#RUNTIME-CONFIG-CONNECTION-SECURITY
ssl_ca_file

Must have cert of trusted authority (`PGDATA=/var/lib/pgsql/data`):

  * `$PGDATA/root.crt` - trusted certificate authorities
  * `$PGDATA/root.crl` - certificates revoked by certificate authorities

in `pg_hba.conf` change auth method to `cert` and add `clientcert=1` on appropriate hostssl lines:


    --- old/pg_hba.conf
    +++ new/pg_hba.conf
    @@...@@
     local rhnsat rhnsat md5
     host  rhnsat rhnsat 127.0.0.1/8 md5
     host  rhnsat rhnsat ::1/128 md5
     local rhnsat postgres ident map=usermap
     local postgres postgres ident map=usermap
     
    -hostssl   rhnsat rhnsat 192.0.2.5/32 md5
    +hostssl   rhnsat rhnsat 192.0.2.5/32 cert clientcert=1

=== Client

In case you want to use client certificates, you need following files present

  * `PGSSLCERT` (defaults to `~/.postgresql/postgresql.crt`)
  * `PGSSLKEY`  (defaults to `~/.postgresql/postgresql.key`)
  * Be sure they have `chmod 0400`.

Client certificate must have `CN` set to desired database username.

==== Python


    import psycopg2
    c = psycopg2.connect("host=... dbname=spacepw user=spaceuser password=spacepass sslmode=verify-full sslcert=/root/.postgresql/aa/postgresql.crt sslkey=/root/.postgresql/aa/postgresql.key").cursor(); c.execute("SELECT label FROM rhnversioninfo")
    c.fetchall()

==== Java

Crutial data probably available here: http://www.postgresql.org/message-id/4C0712E6.1050002@postnewspapers.com.au
(others are well anysearchengineable).

=== Components that needs to be changed

List would be similar to the ones touched while working on bug 1020952.


    $ git log --grep 1020952 --pretty=format:%H | xargs -n1 git diff-tree --no-commit-id --name-only -r | sort -u
    backend/rhn-conf/rhn.conf
    backend/server/rhnSQL/__init__.py
    backend/server/rhnSQL/driver_cx_Oracle.py
    backend/server/rhnSQL/driver_postgresql.py
    backend/server/rhnSQL/sql_base.py
    java/code/src/com/redhat/rhn/common/conf/ConfigDefaults.java
    java/conf/rhn_java.conf
    search-server/spacewalk-search/src/java/com/redhat/satellite/search/db/DatabaseManager.java
    web/modules/rhn/RHN/DBI.pm

On top of that `schema/spacewalk/spacewalk-sql`. (Needs work on SSL as well.)

== Examples

Example script to generate all files you need (even for server side), having
separate certificate for certification authority, server, and client.
Suitable only for testing purposes as no file is password protected.


    LEN=4096
    DAYS=365
    # CA
    openssl req -new -x509 -nodes -out root.crt -keyout root.key \
        -subj /CN=TheRootCA -newkey rsa:$LEN -sha512
    # Server
    openssl req -new -out server.req -keyout server.key -nodes \
        -newkey rsa:$LEN -subj "/CN=$( hostname )/emailAddress=root@localhost"
    openssl x509 -req -in server.req -CAkey root.key -CA root.crt \
        -days $DAYS -set_serial $RANDOM -sha512 -out server.crt
    # Client
    openssl req -new -out postgresql.req -keyout postgresql.key -nodes \
        -newkey rsa:$LEN -subj "/CN=spaceuser"
    openssl x509 -req -in postgresql.req -CAkey root.key -CA root.crt \
        -days $DAYS -set_serial $RANDOM -sha512 -out postgresql.crt

... consider describing solution using our `rhn-ssl-tool`.
