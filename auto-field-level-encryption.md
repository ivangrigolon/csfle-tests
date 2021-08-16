## Test using automatic field level encryption 

### Table of Contents
** [How to git](#how-to-git)**<br>


## How to git 
```
git add .;git commit -m "`date`";git push
```

bash # Using bash
# Creating master key
```
bash
echo "export IVAN_MASTER_LOCAL_KEY=\"$(head -c 96 /dev/urandom | base64 | tr -d '\n')\"" | sudo tee -a ~/.bashrc

source ~/.bashrc
mongosh --nodb --shell --eval "var IVAN_MASTER_LOCAL_KEY='$IVAN_MASTER_LOCAL_KEY'"

keyDB = "keyVaultDB";
keyCOLL = "keyVaultColl";
keyVaultNamespace = keyDB+"."+keyCOLL

//notEncryptedConnection = Mongo("mongodb://localhost:27017/?replicaSet=replset")
//db = notEncryptedConnection.getDB("ivanEncryptDB.ivanEncryptColl").
//coll = db.getCollection("ivanEncryptColl");
// This was already node in previous test so no need to recreate the index.
// notEncryptedConnection.getDB(keyDB).getCollection(keyCOLL).createIndex({keyAltNames:1}, {
    unique: true,
    partialFilterExpression: {
      keyAltNames: {
        $exists: true
      }
    }
  })
```

(Optional )Enforce Field Level Encryption Schema
As explained [here](https://docs.mongodb.com/manual/core/security-client-side-encryption/#enforce-field-level-encryption-schema) you can enforce on the server side that specific fields are encrypted.
Note that this is optional and we are not going to implemet this here. However the below is self-explanatory, remember that the key concept here is that this is a vslidator imposed SERVER SIDE.
```
db.getSiblingDB("hr").runCommand(
  {
    "collMod" : "employees",
    "validator" : {
      "$jsonSchema" : {
        "bsonType" : "object",
        "properties" : {
          "taxid" : {
            "encrypt" : {
              "keyId" : [UUID("e114f7ad-ad7a-4a68-81a7-ebcb9ea0953a")],
              "algorithm" : "AEAD_AES_256_CBC_HMAC_SHA_512-Random",
            }
          },
          "taxid-short" : {
            "encrypt" : {
              "keyId" : [UUID("33408ee9-e499-43f9-89fe-5f8533870617")],
              "algorithm" : "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic",
              "bsonType" : "string"
            }
          }
        }
      }
    }
  }
)
```


### Create the encryption configuration
```
var ivan_ClientSideFieldLevelEncryptionOptions = {"keyVaultNamespace" : keyVaultNamespace,"kmsProviders" : {"local" : {"key" : BinData(0, IVAN_MASTER_LOCAL_KEY)}}}
ivan_csfleDatabaseConnection = Mongo("mongodb://localhost:27017/?replicaSet=replset",ivan_ClientSideFieldLevelEncryptionOptions)

ivankeyVault = ivan_csfleDatabaseConnection.getKeyVault()

IVAN_FIELD1_DATA_KEY = ivankeyVault.createKey( "local",["field1"])
IVAN_FIELD2_DATA_KEY = ivankeyVault.createKey( "local",["field2"])


```
var ivan_ClientSideFieldLevelEncryptionOptions = {
  "keyVaultNamespace" : keyVaultNamespace,
  "kmsProviders" : {
    "local" : {
      "key" : IVAN_MASTER_LOCAL_KEY
    }
  },
  schemaMap : {
    "auto_csfle_db.auto_csfle_coll" : {
      "bsonType": "object",
      "properties" : {
        "field1" : {
          "encrypt" : {
            "keyId" : [IVAN_FIELD1_DATA_KEY],
            "bsonType" : "string",
            "algorithm" : "AEAD_AES_256_CBC_HMAC_SHA_512-Random"
          }
        },
        "field2": {
          "encrypt": {
            "keyId": [IVAN_FIELD2_DATA_KEY],
            "algorithm": "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic",
            "bsonType": "string"
          }
        }
      }
    }
  }
}
```

ivan_csfleDatabaseConnection = Mongo("mongodb://localhost:27017/?replicaSet=replset",ivan_ClientSideFieldLevelEncryptionOptions)

ivan_db_conn_object = ivan_csfleDatabaseConnection.getDB("auto_csfle_db");
ivan_coll_conn_object = ivan_db_conn_object.getCollection("auto_csfle_coll");

ivan_coll_conn_object.insertOne(
  {
    "name" : "Ivan Grigolon",
    "field1" : "This content should be encrypted",
    "field2" : "Also this content should be encrypted",
    "field3" : "This content is not encrypted"
  }
)
```
## Output visible from a connection not loading the master key:

```
Enterprise replset [direct: primary]> db.auto_csfle_coll.find()
[
  {
    _id: ObjectId("611abebc3df9f29bc7b6aac1"),
    name: 'Ivan Grigolon',
    field1: Binary(Buffer.from("0227972db9841746caa5a203b000b51aff0238588895bd437017c4ec9c64ad7c893618f6702236545e3ba38f1398c4a47a15329f882707fe19c91da066c6e8161743134dc8eeeb2f5741c399c364a5a8f5c988ecab695bbda0767b50916db75f74ba1b93ae214facd9dec282e17093ad7a2a", "hex"), 6),
    field2: Binary(Buffer.from("017d73c39f9f5a4955a4eff04a1b125afb028db9962fc36cf8c8c380f6a3bc7754a32acb66e4b853a8664f5d27094533e590992b3dc0b01cf8fb3db20cf57a3fb7a3a988ff2354c207692da2fe20e83b9199296b5fa95ecbbe91d8f39bfc2be74ea3166af43da1caa0aab0b92dc44b9b410c", "hex"), 6),
    field3: 'This content is not encrypted'
  }
]
```

## What to do to read this when you login:
```
bash-3.2$ mongosh --nodb --shell --eval "var IVAN_MASTER_LOCAL_KEY='$IVAN_MASTER_LOCAL_KEY'"

keyDB = "keyVaultDB";

keyCOLL = "keyVaultColl";

keyVaultNamespace = keyDB+"."+keyCOLL

var ivan_ClientSideFieldLevelEncryptionOptions = {"keyVaultNamespace" : keyVaultNamespace,"kmsProviders" : {"local" : {"key" : BinData(0, IVAN_MASTER_LOCAL_KEY)}}}

ivan_csfleDatabaseConnection = Mongo("mongodb://localhost:27017/?replicaSet=replset",ivan_ClientSideFieldLevelEncryptionOptions)

ivan_csfleDatabaseConnection.getDB("auto_csfle_db").getCollection("auto_csfle_coll").find()
```

This should show the output:
```
[
  {
    _id: ObjectId("611abebc3df9f29bc7b6aac1"),
    name: 'Ivan Grigolon',
    field1: 'This content should be encrypted',
    field2: 'Also this content should be encrypted',
    field3: 'This content is not encrypted'
  }
]
```
