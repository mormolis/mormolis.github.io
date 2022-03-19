he problem: We have an application that uses Cassandra as the preferred persistence technology and we would like to have a one-command container running for local development and testing.

In other words, we need the Cassandra container to have the Keyspace and the up-to-date schema following our migration scripts by executing something like  `./run_containerised_cassandra_locally.sh`
Breaking this problem into two sub-problems we identify that in order to do that we need a DB migration tool. In RDBMS we have tools like Flyway or Liquibase. For Cassandra, with a little bit of digging, com.hhandoko:cassandra-migration can be found. Which we can be incorporated into our codebase as a dependency or can exist as a standalone artefact (jar file) to be run on demand by a script. The latest is what we are going to use here.
When we go to releases of that repo, we can see the version of the jar with dependencies. (Current versioncassandra-migration-0.17-jar-with-dependencies.jar).

We want to create a structure like the following:

<img width="810" alt="image" src="https://user-images.githubusercontent.com/18053538/159135993-194b389d-9dbe-4e14-a143-1e03732b38c6.png">

Under schema_migration we place another directory which will hold our migration scripts. The naming should be like the one in the screenshot. Starting with V1__ (mind the double underscore) and a description of the script it contains.

```cql
CREATE TABLE IF NOT exists our_table(
        id text PRIMARY KEY,
        name text,
        team text,
        contact_email text,
        created timestamp,
        enabled boolean
);
```

Now if we want to run the schema migration we need to call the java jar file with some parameters. For the shake of convenience, we introduce the script run.sh that you can see on the screenshot above.

```bash

#!/usr/bin/env bash

java -jar \
-Dcassandra.migration.scripts.locations=$(echo $MIGRATION_SCRIPT_LOCATION) \
-Dcassandra.migration.cluster.contactpoints=$(echo $CASSANDRA_DB_CONTACT_POINTS) \
-Dcassandra.migration.cluster.port=$(echo $CASSANDRA_DB_PORT) \
-Dcassandra.migration.cluster.username=$(echo $CASSANDRA_DB_USERNAME) \
-Dcassandra.migration.cluster.password=$(echo $CASSANDRA_DB_PASSWORD) \
-Dcassandra.migration.keyspace.name=$(echo $KEYSPACE_NAME) \
-Dcassandra.migration.keyspace.consistency=ALL \
-Dcassandra.migration.cluster.enablessl=true \
cassandra-migration-0.17-jar-with-dependencies.jar migrate
```

We need to make it runnable with `chmod +x run.sh`.

And finally, before we run it, we need to export the variables that are passed as system variables to the migration utility. (For local development that there are no really sensitive values We can keep a file with the commands that we will source to our environment, example to follow.

As a bonus, because We are using the migration job as part of our pipeline, we have containerized the migration utility so we can easily use it on Jenkins. You don't have to do the next part if you only want this for the local testing/development container.

```dockerfile
FROM library/openjdk:11.0.2-jre

ADD cassandra-migration-0.17-jar-with-dependencies.jar run.sh /etc/app/

RUN mkdir -p /opt/java/gclogs

ADD migration_scripts /migration_scripts

WORKDIR /etc/app/

EXPOSE 10350

ENTRYPOINT ["/etc/app/run.sh"]
```

Now we are ready to create the Cassandra container with our stuff :D

<img width="667" alt="image" src="https://user-images.githubusercontent.com/18053538/159136137-b988704d-f70e-41b9-9566-ea2e60fb7b66.png">


Now, we are in the local_cassandra directory of the above screenshot. In the dbScripts directory, we just add a simple cql command that will be used to create the keyspace once Cassandra is up and running.

```cql
CREATE KEYSPACE IF NOT EXISTS 
our_keyspace 
WITH replication = {'class':'SimpleStrategy','replication_factor':'1'};
```

Let's create the docker file now:

```dockerfile
FROM cassandra

COPY set_environment_variables /set_environment_variables
COPY entrypoint-wrap.sh /entrypoint-wrap.sh
EXPOSE 9042
ENTRYPOINT ["/entrypoint-wrap.sh"]
CMD ["cassandra", "-f"]
```
On line 3 we copy a file with all variables that the migration script will need. Since we do not have sensitive secrets for the local env we just have the following.

```
export MIGRATION_SCRIPT_LOCATION="filesystem:../schema_migration/migration_scripts" && \
export CASSANDRA_DB_CONTACT_POINTS=localhost && \
export CASSANDRA_DB_PORT=5555 && \
export CASSANDRA_DB_USERNAME="" && \
export CASSANDRA_DB_PASSWORD="" && \
export KEYSPACE_NAME=our_keyspace
export CASSANDRA_DB_SSL_ENABLED=false
```

On line 4 we copy the intermediate script on the container. I call it intermediate because it will run before the real entry point. It is like a wrapper.


```bash
#!/usr/bin/env bash


#to set keyspace name
source set_environment_variables

if [[ ! -z "$KEYSPACE_NAME" && $1 = 'cassandra' ]]; then
  # Create default keyspace for single node cluster
  CQL="CREATE KEYSPACE $KEYSPACE_NAME WITH REPLICATION = {'class': 'SimpleStrategy', 'replication_factor': 1};"
  until echo $CQL | cqlsh; do
    echo "cqlsh: Cassandra is still starting up - please wait - retrying"
    sleep 2
  done && touch /tmp/cassandra_up.flag &
fi

exec /docker-entrypoint.sh "$@"
```

It is definitely an interesting one. Let's analyse its functionality.
On line 5 we source the above file so it can export the variable we set earlier for the migration script. This will make use of the schema name value.
Then it will loop and wait for 2 seconds until the creation of the keyspace will return successfully when it will, it will also create a flag file in `/tmp/cassandra_up.flag` folder that we will use to make sure that Cassandra is up and the keyspace has been created. This file will trigger the schema migration process in the script run_containerised_cassandra_locally.sh.  The & at the end of the line 13 makes the above block of code run asynchronously on the background.

On line 16 we proceed with the real docker entry point passing all the arguments that were initially passed on this script on the Dockerfile.

Finally, our script that will orchestrate all the above and create the Cassandra container.

```bash
#!/usr/bin/env bash

# exc1
echo "------------- cleaning existing instances running ------------- "
docker stop localCassandra
docker container rm localCassandra
# end of exc1

# exc2
echo "------------- running container in detached mode ------------- "
docker build . -t containerised_local_cassandra
docker run --name localCassandra -d -p 5555:9042 containerised_local_cassandra
# end of exc2

printf  "waiting to spin up the containers "

# exc3
while [[ $(docker exec -it localCassandra ls /tmp/cassandra_up.flag 2>&1 | grep "No such file or directory") ]];
do
    sleep 1
    printf .
done
# end of exc3

printf "\n\n------------- setting variables for the migration script: ------------- \n\n"
cat ./set_environment_variables
source ./set_environment_variables

printf "\n\n------------- running the schema migration script -------------\n\n"
java -jar \
-Dcassandra.migration.scripts.locations=$(echo $MIGRATION_SCRIPT_LOCATION) \
-Dcassandra.migration.cluster.contactpoints=$(echo $CASSANDRA_DB_CONTACT_POINTS) \
-Dcassandra.migration.cluster.port=$(echo $CASSANDRA_DB_PORT) \
-Dcassandra.migration.cluster.username=$(echo $CASSANDRA_DB_USERNAME) \
-Dcassandra.migration.cluster.password=$(echo $CASSANDRA_DB_PASSWORD) \
-Dcassandra.migration.keyspace.name=$(echo $KEYSPACE_NAME) \
-Dcassandra.migration.keyspace.consistency=ALL \
-Dcassandra.migration.cluster.enablessl=$(echo $CASSANDRA_DB_SSL_ENABLED) \
../schema_migration/cassandra-migration-0.17-jar-with-dependencies.jar migrate
```

exc1: clean up the already existing Cassandra containers with the name localCassandra so we won't have any conflicts.

exc12 are running the container.

exc3 are waiting for the flag file to be created. This will determine when the container is ready for the next step. 

Finally the Cassandra migration utility (jar) after we source the variables once again.

Don't forget to make it executable!

Now we can have a Cassandra container with our schema and everything just by running
`./run_containerised_cassandra_locally.sh`
