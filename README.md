
# Capture Database Change
#### Developing a reliable/scalable CDC Solution

This architecture is created for purpose of capture database live changes. 
Changes for insertion/updation/deletion or schema changes.


![Architecture](https://i.ibb.co/G2YDMFP/cdc-aws-azure-drawio-3.png)

## Solution Requirement

- Reliable
- Scalable 
- Easy Montioring (in case of issue we can drill down to actual issue)
- Independent of Cloud providor (We can move solution to any other cloud providor)


## Use Case

Our use case was not too complicated, we have micro-services which are doing some
operations. Our application was a project management web application many users using the application
at the same time usally working on same workitems. 

Workitems have a tree like structure there can be child task under a workitem and parent status (inprogress, complete)
is derived from its child tasks. And parent is update be database trigger functions.

If One user do something in workitem it should reflect on realtime for other users.





## Alternatives

There was some other options but i was thinking for solution meet our core requirements.
- [Debezium](https://debezium.io/)
- [Hevo Data]()
- And creating function in database to call webhook directly
- And so on..

## Steps

### Database Changes

- It all start from database setting, Postgres has different wal_level options by default it comes to replica setting
  We have to change it to 'logical replication'.
  - Self provision: If self provisioning database then you can change it from pg_hb.conf & postgres.conf files.
  - AWS RDS: For RDS you have to create new parameter group and change wal_level to 1, then modify database and change parameter group to 
    newly created parameter group.
  - To verify change execute this sql command 
    - postgres> show wal_level; 
    - postgres> show max_replication_slots;
- Creating replication user:  
  - Self provisioning: Create user with replication permission
  - For RDS: Create user and grant permission of repluser as below
    - postgres> create user repluser password 'replpass';  
    - postgres> grant rds_replication to repluser;
- Done with database, Now this user is able to create replication slot and reading wal logs/database changes.

### Listening Database Changes 

- I use python 'psycopg2' library to listening changes. 
- In case of bulk insert/update we also get changes in bulk, Array of changes
```bash
    {
      "change": [
        {
        "kind": "insert",
        "schema": "public",
        "table": "users",
        "columnnames": ["id", "name"],
        "columntypes": ["integer", "character varying(30)"],
        "columnvalues": [111, "Basit Raza"]
        }
      ]
    }
```
- As soon as i get any update i push into my queue. I use Azure Service bus service as queue service.
- Use python 'azure.servicebus' package for queue operations
- Pack working code into container to be hoster on any docker enabled environment
```bash
#Deriving the latest base image
FROM python:3.9

#Labels as key value pair
LABEL Maintainer="CDC"

# Any working directory can be chosen as per choice like '/' or '/home' etc
# i have chosen /
WORKDIR /

COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt

#to COPY the remote file at working directory in container
COPY main.py ./
# Now the structure looks like this '/main.py'

#CMD instruction should be used to run the software
#contained by your image, along with any arguments.

CMD [ "python", "./main.py"]
```

### Listening Queue 

Queue will be listen by different microservice as per there requirements.
For reading queue our implementation is in .net core
[Get started with Azure Service Bus queues (.NET)](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dotnet-get-started-with-queues)
