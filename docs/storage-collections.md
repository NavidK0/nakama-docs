# Collections

Every app or game has data which is specific to the project.

This information must be stored for users, updated, retrieved, and displayed within various parts of a UI. For this purpose the server incorporates a storage engine with a design optimized for [object ownership](storage-access-controls.md), access permissions, and batch operations.

Data is stored in collections with one or more objects which contain a unique key with JSON content. A collection is created without any configuration required. This creates a simple nested namespace which represents the location of a object.

```
Collection
+---------------------------------------------------------------------+
|  Object                                                             |
|  +----------+------------+-------------+-----+-------------------+  |
|  | ?UserId? | Identifier | Permissions | ... |       Value       |  |
|  +---------------------------------------------------------------+  |
+---------------------------------------------------------------------+
```

This design gives great flexibility for developers to group sets of information which belong together within a game or app.

## Write objects

A user can write one or more objects which will be stored in the database server. These objects will be written in a single transaction which guarantees the writes succeed together.

```sh fct_label="cURL"
curl -X PUT "http://127.0.0.1:7350/v2/storage" \
  -H 'Authorization: Bearer <session token>' \
  -d '{"objects":
    [
      {
        "collection": "saves",
        "key": "savegame",
        "value": "{\"progress\": \"50\"}"
      },
      {
        "collection": "stats",
        "key": "skill",
        "value": "{\"progress\": \"24\"}"
      }
    ]
  }'
```

```js fct_label="JavaScript"
var save_game = { "progress": 50 };
var my_stats = { "skill": 24 };
const object_ids = await client.writeStorageObjects(session, [
  {
    "collection": "saves",
    "key": "savegame",
    "value": save_game
  }, {
    "collection": "stats",
    "key": "skills",
    "value": my_stats
  }
]);
console.info("Successfully stored objects: ", object_ids);
```

```csharp fct_label=".NET"
var saveGame = "{ \"progress\": 50 }";
var myStats = "{ \"skill\": 24 }";
var objectIds = await client.WriteStorageObjectsAsync(session, new WriteStorageObject
{
  Collection = "saves",
  Key = "savegame",
  Value = saveGame
}, new WriteStorageObject
{
  Collection = "stats",
  Key = "skills",
  Value = myStats
});
Console.WriteLine("Successfully stored objects: [{0}]", string.Join(",\n  ", objectIds));
```

```csharp fct_label="Unity"
var saveGame = "{ \"progress\": 50 }";
var myStats = "{ \"skill\": 24 }";
var objectIds = await client.WriteStorageObjectsAsync(session, new WriteStorageObject
{
  Collection = "saves",
  Key = "savegame",
  Value = saveGame
}, new WriteStorageObject
{
  Collection = "stats",
  Key = "skills",
  Value = myStats
});
Debug.LogFormat("Successfully stored objects: [{0}]", string.Join(",\n   ", objectIds));
```

```cpp fct_label="Cocos2d-x C++"
auto successCallback = [](const NStorageObjectAcks& acks)
{
  CCLOG("Successfully stored objects %u", acks.size());
};

std::vector<NStorageObjectWrite> objects;
NStorageObjectWrite savesObject, statsObject;
savesObject.collection = "saves";
savesObject.key = "savegame";
savesObject.value = "{ \"progress\": 50 }";
objects.push_back(savesObject);

statsObject.collection = "stats";
statsObject.key = "skills";
statsObject.value = "{ \"skill\": 24 }";
objects.push_back(statsObject);
client->writeStorageObjects(session, objects, successCallback);
```

```js fct_label="Cocos2d-x JS"
var save_game = { "progress": 50 };
var my_stats = { "skill": 24 };
client.writeStorageObjects(session, [
  {
    "collection": "saves",
    "key": "savegame",
    "value": save_game
  }, {
    "collection": "stats",
    "key": "skills",
    "value": my_stats
  }
]).then(function(object_ids) {
      cc.log("Successfully stored objects:", JSON.stringify(object_ids));
    },
    function(error) {
      cc.error("write storage objects failed:", JSON.stringify(error));
    });
```

```cpp fct_label="C++"
auto successCallback = [](const NStorageObjectAcks& acks)
{
  std::cout << "Successfully stored objects " << acks.size() << std::endl;
};

std::vector<NStorageObjectWrite> objects;
NStorageObjectWrite savesObject, statsObject;
savesObject.collection = "saves";
savesObject.key = "savegame";
savesObject.value = "{ \"progress\": 50 }";
objects.push_back(savesObject);

statsObject.collection = "stats";
statsObject.key = "skills";
statsObject.value = "{ \"skill\": 24 }";
objects.push_back(statsObject);
client->writeStorageObjects(session, objects, successCallback);
```

```java fct_label="Java"
String saveGame = "{ \"progress\": 50 }";
String myStats = "{ \"skill\": 24 }";
StorageObjectWrite saveGameObject = new StorageObjectWrite("saves", "savegame", saveGame, PermissionRead.OWNER_READ, PermissionWrite.OWNER_WRITE);
StorageObjectWrite statsObject = new StorageObjectWrite("stats", "skills", myStats, PermissionRead.OWNER_READ, PermissionWrite.OWNER_WRITE);
StorageObjectAcks acks = client.writeStorageObjects(session, saveGameObject, statsObject).get();
System.out.format("Stored objects %s", acks.getAcksList());
```

```swift fct_label="Swift"
// Requires Nakama 1.x

let saveGame = "{\"progress\": 50}".data(using: .utf8)!
let myStats = "{\"skill\": 24}".data(using: .utf8)!

let bucket = "myapp"
var message = StorageWriteMessage()
message.write(bucket: bucket, collection: "saves", key: "savegame", value: saveGame)
message.write(bucket: bucket, collection: "saves", key: "mystats", value: myStats)
client.send(message: message).then { list in
  for recordId in list {
    NSLog("Stored record has version '%@'", recordId.version)
  }
}.catch { err in
  NSLog("Error %@ : %@", err, (err as! NakamaError).message)
}
```

```fct_label="REST"
PUT /v2/storage
Host: 127.0.0.1:7350
Accept: application/json
Content-Type: application/json
Authorization: Bearer <session token>

{
  "objects": [
    {
      "collection": "saves",
      "key": "key",
      "value": "{\"hello\": \"world\"}"
    },
    {
      "collection": "stats",
      "key": "skill",
      "value": "{\"progress\": \"24\"}"
    }
  ]
}
```

### Conditional writes

When objects are successfully stored a version is returned which can be used with further updates to perform concurrent modification checks with the next write. This is known as a conditional write.

A conditional write ensures a client can only update the object if they've seen the previous version of the object. The purpose is to prevent a change to the object if another client has changed the value between when the first client's read and it's next write.

```sh fct_label="cURL"
curl -X PUT "http://127.0.0.1:7350/v2/storage" \
  -H 'Authorization: Bearer <session token>' \
  -d '{
    "objects": [
      {
        "collection": "saves",
        "key": "savegame",
        "value": "{\"progress\": \"50\"}",
        "version": "some-previous-version"
      }
    ]
  }'
```

```js fct_label="JavaScript"
var save_game = { "progress": 50 };
const object_ids = await client.writeStorageObjects(session, [
  {
    "collection": "saves",
    "key": "savegame",
    "value": save_game,
    "version": "<version>"
  }
]);
console.info("Stored objects: %o", object_ids);
```

```csharp fct_label=".NET"
var saveGame = "{ \"progress\": 50 }";
var objectIds = await client.WriteStorageObjectsAsync(session, new WriteStorageObject
{
  Collection = "saves",
  Key = "savegame",
  Value = saveGame,
  Version = "<version>"
});
Console.WriteLine("Stored objects: [{0}]", string.Join(",\n  ", objectIds));
```

```csharp fct_label="Unity"
var saveGame = "{ \"progress\": 50 }";
var objectIds = await client.WriteStorageObjectsAsync(session, new WriteStorageObject
{
  Collection = "saves",
  Key = "savegame",
  Value = saveGame,
  Version = "<version>"
});
Debug.LogFormat("Stored objects: [{0}]", string.Join(",\n  ", objectIds));
```

```cpp fct_label="Cocos2d-x C++"
auto successCallback = [](const NStorageObjectAcks& acks)
{
  CCLOG("Successfully stored objects %u", acks.size());
};

std::vector<NStorageObjectWrite> objects;
NStorageObjectWrite savesObject;
savesObject.collection = "saves";
savesObject.key = "savegame";
savesObject.value = "{ \"progress\": 50 }";
savesObject.version = "<version>";
objects.push_back(savesObject);
client->writeStorageObjects(session, objects, successCallback);
```

```js fct_label="Cocos2d-x JS"
var save_game = { "progress": 50 };
client.writeStorageObjects(session, [
  {
    "collection": "saves",
    "key": "savegame",
    "value": save_game,
    "version": "<version>"
  }
]).then(function(object_ids) {
      cc.log("Stored objects:", JSON.stringify(object_ids));
    },
    function(error) {
      cc.error("write storage objects failed:", JSON.stringify(error));
    });
```

```cpp fct_label="C++"
auto successCallback = [](const NStorageObjectAcks& acks)
{
  std::cout << "Successfully stored objects " << acks.size() << std::endl;
};

std::vector<NStorageObjectWrite> objects;
NStorageObjectWrite savesObject;
savesObject.collection = "saves";
savesObject.key = "savegame";
savesObject.value = "{ \"progress\": 50 }";
savesObject.version = "<version>";
objects.push_back(savesObject);
client->writeStorageObjects(session, objects, successCallback);
```

```java fct_label="Java"
String saveGame = "{ \"progress\": 50 }";
StorageObjectWrite object = new StorageObjectWrite("saves", "savegame", saveGame, PermissionRead.OWNER_READ, PermissionWrite.OWNER_WRITE);
object.setVersion("<version>");
StorageObjectAcks acks = client.writeStorageObjects(session, object).get();
System.out.format("Stored objects %s", acks.getAcksList());
```

```swift fct_label="Swift"
// Requires Nakama 1.x

let saveGame = "{\"progress\": 54}".data(using: .utf8)!
var version = record.version // a StorageRecordId object's version.

var message = StorageWriteMessage()
message.write(bucket: "myapp", collection: "saves", key: "savegame", value: saveGame, version: version)
client.send(message: message).then { list in
  version = list[0].version // Cache updated version for next write.
}.catch { err in
  NSLog("Error %@ : %@", err, (err as! NakamaError).message)
}
```

```fct_label="REST"
PUT /v2/storage
Host: 127.0.0.1:7350
Accept: application/json
Content-Type: application/json
Authorization: Bearer <session token>

{
  "objects": [
    {
      "collection": "saves",
      "key": "key",
      "value": "{\"hello\": \"world\"}",
      "version": "<version>"
    }
  ]
}
```

We support another kind of conditional write which is used to write an object only if none already exists for that object's collection and key.

```sh fct_label="cURL"
curl -X PUT "http://127.0.0.1:7350/v2/storage" \
  -H 'Authorization: Bearer <session token>' \
  -d '{
    "objects": [
      {
        "collection": "saves",
        "key": "savegame",
        "value": "{\"progress\": \"50\"}",
        "version": "*"
      }
    ]
  }'
```

```js fct_label="JavaScript"
var save_game = { "progress": 50 };
const object_ids = await client.writeStorageObjects(session, [
  {
    "collection": "saves",
    "key": "savegame",
    "value": save_game,
    "version": "*"
  }
]);
console.info("Stored objects: %o", object_ids);
```

```csharp fct_label=".NET"
var saveGame = "{ \"progress\": 50 }";
var objectIds = await client.WriteStorageObjectsAsync(session, new WriteStorageObject
{
  Collection = "saves",
  Key = "savegame",
  Value = saveGame,
  Version = "*"
});
Console.WriteLine("Stored objects: [{0}]", string.Join(",\n  ", objectIds));
```

```csharp fct_label="Unity"
var saveGame = "{ \"progress\": 50 }";
var objectIds = await client.WriteStorageObjectsAsync(session, new WriteStorageObject
{
  Collection = "saves",
  Key = "savegame",
  Value = saveGame,
  Version = "*"
});
Debug.LogFormat("Stored objects: [{0}]", string.Join(",\n  ", objectIds));
```

```cpp fct_label="Cocos2d-x C++"
auto successCallback = [](const NStorageObjectAcks& acks)
{
  CCLOG("Successfully stored objects %u", acks.size());
};

std::vector<NStorageObjectWrite> objects;
NStorageObjectWrite savesObject;
savesObject.collection = "saves";
savesObject.key = "savegame";
savesObject.value = "{ \"progress\": 50 }";
savesObject.version = "*";
objects.push_back(savesObject);
client->writeStorageObjects(session, objects, successCallback);
```

```js fct_label="Cocos2d-x JS"
var save_game = { "progress": 50 };
client.writeStorageObjects(session, [
  {
    "collection": "saves",
    "key": "savegame",
    "value": save_game,
    "version": "*"
  }
]).then(function(object_ids) {
      cc.log("Stored objects:", JSON.stringify(object_ids));
    },
    function(error) {
      cc.error("write storage objects failed:", JSON.stringify(error));
    });
```

```cpp fct_label="C++"
auto successCallback = [](const NStorageObjectAcks& acks)
{
  std::cout << "Successfully stored objects " << acks.size() << std::endl;
};

std::vector<NStorageObjectWrite> objects;
NStorageObjectWrite savesObject;
savesObject.collection = "saves";
savesObject.key = "savegame";
savesObject.value = "{ \"progress\": 50 }";
savesObject.version = *";
objects.push_back(savesObject);
client->writeStorageObjects(session, objects, successCallback);
```

```java fct_label="Java"
String saveGame = "{ \"progress\": 50 }";
StorageObjectWrite object = new StorageObjectWrite("saves", "savegame", saveGame, PermissionRead.OWNER_READ, PermissionWrite.OWNER_WRITE);
object.setVersion("*");
StorageObjectAcks acks = client.writeStorageObjects(session, object).get();
System.out.format("Stored objects %s", acks.getAcksList());
```

```java fct_label="Android/Java"
// Requires Nakama 1.x

String saveGame = "{\"progress\": 1}";
String version = "*"; // represents "no version".

CollatedMessage<ResultSet<RecordId>> message = StorageWriteMessage.Builder.newBuilder()
    .record("myapp", "saves", "savegame", saveGame, version)
    .build();
Deferred<ResultSet<RecordId>> deferred = client.send(message);
deferred.addCallback(new Callback<ResultSet<RecordId>, ResultSet<RecordId>>() {
  @Override
  public ResultSet<RecordId> call(ResultSet<RecordId> list) throws Exception {
    // Cache updated version for next write.
    version = list.getResults().get(0).getVersion();
    return list;
  }
}).addErrback(new Callback<Error, Error>() {
  @Override
  public Error call(Error err) throws Exception {
    System.err.format("Error('%s', '%s')", err.getCode(), err.getMessage());
    return err;
  }
});
```

```swift fct_label="Swift"
// Requires Nakama 1.x

let saveGame = "{\"progress\": 1}".data(using: .utf8)!
var version = "*" // represents "no version".

var message = StorageWriteMessage()
message.write(bucket: "myapp", collection: "saves", key: "savegame", value: saveGame, version: version)
client.send(message: message).then { list in
  version = list[0].version // Cache updated version for next write.
}.catch { err in
  NSLog("Error %@ : %@", err, (err as! NakamaError).message)
}
```

```fct_label="REST"
PUT /v2/storage
Host: 127.0.0.1:7350
Accept: application/json
Content-Type: application/json
Authorization: Bearer <session token>

{
  "objects": [
    {
      "collection": "saves",
      "key": "key",
      "value": "{\"hello\": \"world\"}",
      "version": "*"
    }
  ]
}
```

## Read objects

Just like with [writing objects](#write-objects) you can read one or more objects from the database server.

Each object has an owner and permissions. An object can only be read if the permissions allow it. An object which has no owner can be fetched with `"null"` and is useful for global objects which all users should be able to read.

```sh fct_label="cURL"
curl -X POST "http://127.0.0.1:7350/v2/storage" \
  -H 'Authorization: Bearer <session token>' \
  -d '{
    "object_ids": [
      {
        "collection": "saves",
        "key": "savegame",
        "user_id": "some-user-id"
      }
    ]
  }'
```

```js fct_label="JavaScript"
const objects = await client.readStorageObjects(session, {
  "object_ids": [{
    "collection": "saves",
    "key": "savegame",
    "user_id": session.user_id
  }]
});
console.info("Read objects: %o", objects);
```

```csharp fct_label=".NET"
var result = await client.ReadStorageObjectsAsync(session, new StorageObjectId {
  Collection = "saves",
  Key = "savegame",
  UserId = session.UserId
});
Console.WriteLine("Read objects: [{0}]", string.Join(",\n  ", result.Objects));
```

```csharp fct_label="Unity"
var result = await client.ReadStorageObjectsAsync(session, new StorageObjectId {
  Collection = "saves",
  Key = "savegame",
  UserId = session.UserId
});
Debug.LogFormat("Read objects: [{0}]", string.Join(",\n  ", result.Objects));
```

```cpp fct_label="Cocos2d-x C++"
auto successCallback = [](const NStorageObjects& objects)
{
  for (auto& object : objects)
  {
    CCLOG("Object key: %s, value: %s", object.key.c_str(), object.value.c_str());
  }
};
std::vector<NReadStorageObjectId> objectIds;
NReadStorageObjectId objectId;
objectId.collection = "saves";
objectId.key = "savegame";
objectId.userId = session->getUserId();
objectIds.push_back(objectId);
client->readStorageObjects(session, objectIds, successCallback);
```

```js fct_label="Cocos2d-x JS"
client.readStorageObjects(session, {
  "object_ids": [{
    "collection": "saves",
    "key": "savegame",
    "user_id": session.user_id
  }]
}).then(function(objects) {
      cc.log("Read objects:", JSON.stringify(objects));
    },
    function(error) {
      cc.error("read storage objects failed:", JSON.stringify(error));
    });
```

```cpp fct_label="C++"
auto successCallback = [](const NStorageObjects& objects)
{
  for (auto& object : objects)
  {
    std::cout << "Object key: " << object.key << ", value: " << object.value << std::endl;
  }
};
std::vector<NReadStorageObjectId> objectIds;
NReadStorageObjectId objectId;
objectId.collection = "saves";
objectId.key = "savegame";
objectId.userId = session->getUserId();
objectIds.push_back(objectId);
client->readStorageObjects(session, objectIds, successCallback);
```

```java fct_label="Java"
StorageObjectId objectId = new StorageObjectId("saves");
objectId.setKey("savegame");
objectId.setUserId(session.getUserId());
StorageObjects objects = client.readStorageObjects(session, objectId).get();
System.out.format("Read objects %s", objects.getObjectsList().toString());
```

```swift fct_label="Swift"
// Requires Nakama 1.x

let userID = session.userID // a Session object's Id.

var message = StorageFetchMessage()
message.fetch(bucket: "myapp", collection: "saves", key: "savegame", userID: userID)
message.fetch(bucket: "myapp", collection: "configuration", key: "config", userID: nil)
client.send(message: message).then { list in
  for record in list {
    NSLog("Record value '%@'", record.value)
    NSLog("Record permissions read '%@' write '%@'",
        record.permissionRead, record.permissionWrite)
  }
}.catch { err in
  NSLog("Error %@ : %@", err, (err as! NakamaError).message)
}
```

```fct_label="REST"
POST /v2/storage
Host: 127.0.0.1:7350
Accept: application/json
Content-Type: application/json
Authorization: Bearer <session token>

{
  "object_ids": [
    {
      "collection": "saves",
      "key": "savegame",
      "user_id": "some-user-id"
    }
  ]
}
```

## List objects

You can list objects in a collection and page through results. The objects returned can be filter to those owned by the user or `"null"` for public records which aren't owned by a user.

```sh fct_label="cURL"
curl -X GET "http://127.0.0.1:7350/v2/storage/saves?user_id=some-user-id&limit=10" \
  -H 'Authorization: Bearer <session token>'
```

```js fct_label="JavaScript"
const limit = 100; // default is 10.
const objects = await client.listStorageObjects(session, "saves", session.user_id, limit);
console.info("List objects: %o", objects);
```

```csharp fct_label=".NET"
const int limit = 100; // default is 10.
var result = await client.ListUsersStorageObjectsAsync(session, "saves", session.UserId, limit);
Console.WriteLine("List objects: {0}", result);
```

```csharp fct_label="Unity"
const int limit = 100; // default is 10.
var result = await client.ListUsersStorageObjectsAsync(session, "saves", session.UserId, limit);
Debug.LogFormat("List objects: {0}", result);
```

```cpp fct_label="Cocos2d-x C++"
auto successCallback = [](NStorageObjectListPtr list)
{
  for (auto& object : list->objects)
  {
    CCLOG("Object key: %s, value: %s", object.key.c_str(), object.value.c_str());
  }
};
client->listUsersStorageObjects(session,
    "saves",
    session->getUserId(),
    opt::nullopt,
    opt::nullopt,
    successCallback);
```

```js fct_label="Cocos2d-x JS"
client.listStorageObjects(session, "saves", session.user_id)
  .then(function(objects) {
      cc.log("List objects:", JSON.stringify(objects));
    },
    function(error) {
      cc.error("list storage objects failed:", JSON.stringify(error));
    });
```

```cpp fct_label="C++"
auto successCallback = [](NStorageObjectListPtr list)
{
  for (auto& object : list->objects)
  {
    std::cout << "Object key: " << object.key << ", value: " << object.value << std::endl;
  }
};
client->listUsersStorageObjects(session,
    "saves",
    session->getUserId(),
    opt::nullopt,
    opt::nullopt,
    successCallback);
```

```java fct_label="Java"
StorageObjectList objects = client.listUsersStorageObjects(session, "saves", session.getUserId()).get();
System.out.format("List objects %s", objects);
```

```swift fct_label="Swift"
// Requires Nakama 1.x

let userId = session.userID // a Session object's Id.

var message = StorageListMessage(bucket: "myapp")
message.collection = "saves"
message.userID = userId
client.send(message: message).then { list in
  for record in list {
    NSLog("Record value '%@'", record.value)
    NSLog("Record permissions read '%@' write '%@'",
        record.permissionRead, record.permissionWrite)
  }
}.catch { err in
  NSLog("Error %@ : %@", err, (err as! NakamaError).message)
}
```

```fct_label="REST"
GET /v2/storage/<collection>?user_id=<user_id>&limit=<limit>&cursor=<cursor>
Host: 127.0.0.1:7350
Accept: application/json
Content-Type: application/json
Authorization: Bearer <session token>
```

## Remove objects

A user can remove an object if it has the correct permissions and they own it.

```sh fct_label="cURL"
curl -X PUT "http://127.0.0.1:7350/v2/storage/delete" \
  -H 'Authorization: Bearer <session token>' \
  -d '{
    "object_ids": [
      {
        "collection": "saves",
        "key": "savegame"
      }
    ]
  }'
```

```js fct_label="JavaScript"
await client.deleteStorageObjects(session, {
  "object_ids": [{
    "collection": "saves",
    "key": "savegame"
  }]
});
console.info("Deleted objects.");
```

```csharp fct_label=".NET"
var result = await client.DeleteStorageObjectsAsync(session, new StorageObjectId {
  Collection = "saves",
  Key = "savegame"
});
Console.WriteLine("Deleted objects.");
```

```csharp fct_label="Unity"
var result = await client.DeleteStorageObjectsAsync(session, new StorageObjectId {
  Collection = "saves",
  Key = "savegame"
});
Debug.Log("Deleted objects.");
```

```cpp fct_label="Cocos2d-x C++"
auto successCallback = []()
{
  CCLOG("Deleted objects.");
};

std::vector<NDeleteStorageObjectId> objectIds;
NDeleteStorageObjectId objectId;
objectId.collection = "saves";
objectId.key = "savegame";
objectIds.push_back(objectId);
client->deleteStorageObjects(session, objectIds, successCallback);
```

```js fct_label="Cocos2d-x JS"
client.deleteStorageObjects(session, {
  "object_ids": [{
    "collection": "saves",
    "key": "savegame"
  }]
}).then(function() {
      cc.log("Deleted objects.");
    },
    function(error) {
      cc.error("delete storage objects failed:", JSON.stringify(error));
    });
```

```cpp fct_label="C++"
auto successCallback = []()
{
  std::cout << "Deleted objects." << std::endl;
};

std::vector<NDeleteStorageObjectId> objectIds;
NDeleteStorageObjectId objectId;
objectId.collection = "saves";
objectId.key = "savegame";
objectIds.push_back(objectId);
client->deleteStorageObjects(session, objectIds, successCallback);
```

```java fct_label="Java"
StorageObjectId objectId = new StorageObjectId("saves");
objectId.setKey("savegame");
client.deleteStorageObjects(session, objectId).get();
System.out.format("Deleted objects.");
```

```swift fct_label="Swift"
// Requires Nakama 1.x

var message = StorageRemoveMessage()
message.remove(bucket: "myapp", collection: "saves", key: "savegame")
client.send(message: message).then {
  NSLog("Removed user's record(s).")
}.catch { err in
  NSLog("Error %@ : %@", err, (err as! NakamaError).message)
}
```

```fct_label="REST"
PUT /v2/storage/delete
Host: 127.0.0.1:7350
Accept: application/json
Content-Type: application/json
Authorization: Bearer <session token>

{
  "object_ids": [
    {
      "collection": "saves",
      "key": "savegame"
    }
  ]
}
```

You can also conditionally remove an object if the object version matches the version sent by the client.

```sh fct_label="cURL"
curl -X PUT \
  http://127.0.0.1:7350/v2/storage/delete \
  -H 'Authorization: Bearer <session token>' \
  -d '{
    "object_ids": [
      {
        "collection": "saves",
        "key": "savegame",
        "version": "<version>"
      }
    ]
  }'
```

```js fct_label="JavaScript"
await client.deleteStorageObjects(session, {
  "object_ids": [{
    "collection": "saves",
    "key": "savegame",
    "version": "<version>"
  }]
});
console.info("Deleted objects.");
```

```csharp fct_label=".NET"
var result = await client.DeleteStorageObjectsAsync(session, new StorageObjectId {
  Collection = "saves",
  Key = "savegame",
  UserId = session.UserId,
  Version = "<version>"
});
Console.WriteLine("Deleted objects.");
```

```csharp fct_label="Unity"
var result = await client.DeleteStorageObjectsAsync(session, new StorageObjectId {
  Collection = "saves",
  Key = "savegame",
  UserId = session.UserId,
  Version = "<version>"
});
Debug.Log("Deleted objects.");
```

```cpp fct_label="Cocos2d-x C++"
auto successCallback = []()
{
  CCLOG("Deleted objects.");
};

std::vector<NDeleteStorageObjectId> objectIds;
NDeleteStorageObjectId objectId;
objectId.collection = "saves";
objectId.key = "savegame";
objectId.version = "<version>";
objectIds.push_back(objectId);
client->deleteStorageObjects(session, objectIds, successCallback);
```

```js fct_label="Cocos2d-x JS"
client.deleteStorageObjects(session, {
  "object_ids": [{
    "collection": "saves",
    "key": "savegame",
    "version": "<version>"
  }]
}).then(function() {
      cc.log("Deleted objects.");
    },
    function(error) {
      cc.error("delete storage objects failed:", JSON.stringify(error));
    });
```

```cpp fct_label="C++"
auto successCallback = []()
{
  std::cout << "Deleted objects." << std::endl;
};

std::vector<NDeleteStorageObjectId> objectIds;
NDeleteStorageObjectId objectId;
objectId.collection = "saves";
objectId.key = "savegame";
objectId.version = "<version>";
objectIds.push_back(objectId);
client->deleteStorageObjects(session, objectIds, successCallback);
```

```java fct_label="Java"
StorageObjectId objectId = new StorageObjectId("saves");
objectId.setKey("savegame");
objectId.setVersion("<version>");
client.deleteStorageObjects(session, objectId).get();
System.out.format("Deleted objects.");
```

```swift fct_label="Swift"
// Requires Nakama 1.x

let version = record.version // a StorageRecordId object's version.

var message = StorageRemoveMessage()
message.remove(bucket: "myapp", collection: "saves", key: "savegame", version: version)
client.send(message: message).then {
  NSLog("Removed user's record(s).")
}.catch { err in
  NSLog("Error %@ : %@", err, (err as! NakamaError).message)
}
```

```fct_label="REST"
PUT /v2/storage/delete
Host: 127.0.0.1:7350
Accept: application/json
Content-Type: application/json
Authorization: Bearer <session token>

{
  "object_ids": [
    {
      "collection": "saves",
      "key": "savegame",
      "version": "<version>"
    }
  ]
}
```

