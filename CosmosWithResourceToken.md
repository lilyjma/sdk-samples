## Accessing Data in Cosmos DB Using Resource Token

This sample demonstrates how to get and use a resource token to access certain items in a container inside a database. 
  
You can use a resource token (by creating Cosmos DB users and permissions) when you want to provide access to resources in your Cosmos DB account to a client that cannot be trusted with the master key.

If you'd like more information, please see ["Secure access to data in Azure Cosmos DB"](https://docs.microsoft.com/en-us/azure/cosmos-db/secure-access-to-data).

Note that this sample uses the Python Cosmos SDK version 4.0.0b6. 



### Get Resource Token
```
# 0. Imports
import os
from azure.cosmos import exceptions, CosmosClient, PartitionKey
import azure.cosmos.documents as documents

# 1. Connect to your database (create one if necessary)

url = os.environ["ACCOUNT_URI"]
key = os.environ["MASTER_KEY"]
client = CosmosClient(url, key)

database_name = "my_database"

try: 
    database = client.create_database(database_name)
except exceptions.CosmosResourceExistsError:
    database = client.get_database_client(database_name)


# 2. Get container with item(s) that you want access to (create one if necessary)

container_name = "my_container"
try: 
    container = database.create_container(id=container_name, partition_key=PartitionKey(path="/key"))
except exceptions.CosmosResourceExistsError:
    container = database.get_container_client(container_name)

# 3. Create a user 
user_name = "Test User"
try: 
    user = database.create_user(body={"id": user_name})
except exceptions.CosmosHttpResponseError:
    user = database.get_user_client(user_name)

# 4. Give this user permission to access items with PartitionKey = "1" in container 

permission_name = "my_all_permission"

permission_definition = {
    "id" : permission_name,
    "permissionMode": documents.PermissionMode.All,
    "resource": container.container_link,
    "resourcePartitionKey": ["1"]
}

try: 
    permission = user.create_permission(permission_definition)
except exceptions.CosmosHttpResponseError: 
    permission = user.get_permission(permission_name)

# 5. Get the resource token that's associated with the permission created

token = {}
token[container.id] = permission.properties["_token"]

```

At this point, we've created *Test User* who has access to items with partition key "1" inside *my_container*, which is a container inside *my_database*. This user was given the *ALL* permission, which allows READ, UPDATE, and DELETE operations. The other options are: 

  * PermissionMode.Read: READ operations only 
  * PermissionMode.NoneMode: None

The resource token comes with the created permission can be used to a create a CosmosClient who has access to data specified by the permission definition. 


### Use Resource Token to Access Data 

Now pretend that you're given this token, which gives you access to only specific data from *my_container* in *my_database*:

```
token_client = CosmosClient(url, token)
database = token_client.get_database_client("my_database")
container = database.get_container_client("my_container")


# Pretend that "item1" the container has PartitionKey "1" 

# Read 
try: 
    item_1 = container.read_item(item="item1", partition_key="1")
    print(item_1)
except exceptions.CosmosHttpResponseError: 
    print("Error in reading item")


# Update
try: 
    container.upsert_item(
        {
            "id":"item1",
            "key":"1",
            "info":"updating info"
        }
    )
except exceptions.CosmosHttpResponseError: 
    print("Error in updating item")


# Delete 
try:
    container.delete_item(item="item1", partition_key="1")
except exceptions.CosmosHttpResponseError: 
    print("Error in deleting item")


# Create 
try:
    container.create_item(item="new_item1", partition_key="1")
except exceptions.CosmosHttpResponseError: 
    print("Error in creating item")

# Query 
try:
    for item in container.query_items(
        query='SELECT * FROM my_container c WHERE c.key= "1"',
        partition_key="1",
    ):
        print(json.dumps(item, indent=True))
except exceptions.CosmosHttpResponseError: 
    print("Error in querying item(s)")
```
    


















