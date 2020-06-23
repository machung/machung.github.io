# How to backup and restore MongoDb database

## Backup database

```sh
# ssh to mongo db server
ssh user@your.db.server.ip

# dump database
# it will generate a dump file in the current workspace (/home/user).
mongodump --archive=mydb-archive -d=mydb -u=admin-account -p=admin-account-pwd --authenticationDatabase=admin
```

## Restore database into docker container

```sh
# back to Windows
# copy dump folder from db server
scp user@your.db.server.ip:/home/user/mydb-archive .

# copy dump folder into container
# db-container is the container name
docker cp mydb-archive db-container:/home/

# run container bash
docker exec -it db-container bash

# if wanna restore to different database name, use --nsFrom and --nsTo
cd /home
mongorestore --archive=mydb-archive -u=admin-account -p=admin-account-pwd --authenticationDatabase=admin --nsFrom="mydb.*" --nsTo="mydb-diff.*"
```
