
# Capture Database Change
(A Hybrid Cloud Approach)
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
FROM python:3.8

WORKDIR /

COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt

COPY main.py ./

ARG DB_CONNECTION_STRING
ENV DB_CONNECTION_STRING=$DB_CONNECTION_STRING

RUN echo "DB_CONNECTION_STRING= $DB_CONNECTION_STRING"

ARG SERVICE_BUS_CONNECTION_STRING
ENV SERVICE_BUS_CONNECTION_STRING=${SERVICE_BUS_CONNECTION_STRING}
RUN echo "SERVICE_BUS_CONNECTION_STRING= $SERVICE_BUS_CONNECTION_STRING"

ARG SERVICE_BUS_QUEUE
ENV SERVICE_BUS_QUEUE=${SERVICE_BUS_QUEUE}
RUN echo "SERVICE_BUS_QUEUE= $SERVICE_BUS_QUEUE"

ARG REPLICATION_SLOT_NAME
ENV REPLICATION_SLOT_NAME=${REPLICATION_SLOT_NAME}
RUN echo "REPLICATION_SLOT_NAME= $REPLICATION_SLOT_NAME"



CMD [ "python", "./main.py"]
```

### Listening Queue 

Queue will be listen by different microservice as per there requirements.
For reading queue our implementation is in .net core
[Get started with Azure Service Bus queues (.NET)](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dotnet-get-started-with-queues)
## Second Propose Solution
Before the above solution i have propose this solution but later come with above better solution.

This was using AWS kinesis datastream as queue and AWS Lambda for CDC
![First Solution](https://i.ibb.co/QHvD3SL/Screenshot-2022-04-23-at-4-50-53-PM.png")

### Whats wrong here?
There was several annoying things that lead me to think for other solution.
- First we heavily depend on AWS (Lambda, RDS, Kinesis) it would be a bottleneck if we ever
  think to change our cloud providor.
- Create a new IAM user for kinesis permissions. Although it not bad but an extra step, Have to provide kinesis full permissions BTW.
- For pushing messages to kinesis stream we have to use [boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html) And setting up env variable for AWS sdk.
- Consuming kinesis stream was a mess, its client library is written in Java, But they provide a [Multilandaemon]() Which we can use to write code in any lang.
- [Multilangdaemon]() require [DynamoDb]() full access as this is use for store lease and checkpoint etc.
- These steps have to replicate by each developer who is working on this task this would be a mess

These are some draw backs for Kinesis in my case for which i move to other queue option. Choosing Service bus is a reason because we planning to move to azure in near future. 

