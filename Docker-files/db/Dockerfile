FROM mysql:8.0.33

LABEL "Project"="Vprofile"
LABEL "Author"="Ujjwal"
ENV MYSQL_ROOT_PASSWORD="vprodbpass"
ENV MYSQL_DATABASE="accounts"

ADD db_backup.sql docker-entrypoint-initdb/db_backup.sql