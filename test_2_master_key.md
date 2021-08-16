## Test of using 2 different master key 

### Table of Contents
** [How to git](#how-to-git)**<br>


## How to git 
```
`git add .;git commit -m "`date`";git push`
```

bash # Using bash
# Create the master-key
```
bash
echo "export TOM_LOCAL_KEY=\"$(head -c 96 /dev/urandom | base64 | tr -d '\n')\"" >> ~/.profile
echo "export TIM_LOCAL_KEY=\"$(head -c 96 /dev/urandom | base64 | tr -d '\n')\"" >> ~/.profile
source ~/.profile
mongosh --nodb --shell --eval "var TIM_LOCAL_KEY='$TIM_LOCAL_KEY'; var TOM_LOCAL_KEY='$TOM_LOCAL_KEY'"

keyDB = "keyVaultDB";
keyCOLL = "keyVaultColl";
keyVaultNamespace = keyDB+"."+keyCOLL
notEncryptedConnection = Mongo("mongodb://localhost:27017/?replicaSet=replset")

//db = notEncryptedConnection.getDB("ivanEncryptDB.ivanEncryptColl").
//coll = db.getCollection("ivanEncryptColl");

notEncryptedConnection.getDB(keyDB).getCollection(keyCOLL).createIndex({keyAltNames:1}, {
    unique: true,
    partialFilterExpression: {
      keyAltNames: {
        $exists: true
      }
    }
  })


## Create the encryption configuration
```
var tom_ClientSideFieldLevelEncryptionOptions = {"keyVaultNamespace" : keyVaultNamespace,"kmsProviders" : {"local" : {"key" : BinData(0, TOM_LOCAL_KEY)}}}
var tim_ClientSideFieldLevelEncryptionOptions = {"keyVaultNamespace" : keyVaultNamespace,"kmsProviders" : {"local" : {"key" : BinData(0, TIM_LOCAL_KEY)}}}

tom_csfleDatabaseConnection = Mongo("mongodb://localhost:27017/?replicaSet=replset",tom_ClientSideFieldLevelEncryptionOptions)
tim_csfleDatabaseConnection = Mongo("mongodb://localhost:27017/?replicaSet=replset",tim_ClientSideFieldLevelEncryptionOptions)

## Create the Key Vault object in order to access the methods to create the keys
```
tom_keyVault = tom_csfleDatabaseConnection.getKeyVault();
tim_keyVault = tim_csfleDatabaseConnection.getKeyVault();
```
## Create the Key if not created already or otherwise get them by name

TOM_DATA_KEY = tom_keyVault.createKey( "local",["tomKey"])
TIM_DATA_KEY = tim_keyVault.createKey( "local",["timKey"])
```


# Let's store the KEY we want (optional if you have created them in the step before)

//TOM_DATA_KEY = notEncryptedConnection.getDB(keyDB).getCollection(keyCOLL).findOne({ keyAltNames: { $in: ["tomKey"] } })._id
//TIM_DATA_KEY = notEncryptedConnection.getDB(keyDB).getCollection(keyCOLL).findOne({ keyAltNames: { $in: ["timKey"] } })._id


#Now that we get the key we can connect with an encrypted client
tomEncryptedClient = Mongo("mongodb://localhost:27017/?replicaSet=replset",tom_ClientSideFieldLevelEncryptionOptions)
timEncryptedClient = Mongo("mongodb://localhost:27017/?replicaSet=replset",tim_ClientSideFieldLevelEncryptionOptions)


tomkeyVault = tomEncryptedClient.getKeyVault()
timkeyVault = timEncryptedClient.getKeyVault() 

tomkeyVault.getKeyByAltName("tomKey")
timkeyVault.getKeyByAltName("timKey")

tom_db_conn_object = tomEncryptedClient.getDB("csfle_db");
tom_coll_conn_object = tom_db_conn_object.getCollection("csfle_coll");
tim_db_conn_object = tomEncryptedClient.getDB("csfle_db");
tim_coll_conn_object = tim_db_conn_object.getCollection("csfle_coll");

tomclientEncryption = tomEncryptedClient.getClientEncryption()
timclientEncryption = timEncryptedClient.getClientEncryption()

tom_coll_conn_object.insertOne({
  "name" : "Tom",
  "comment" : "I have encrypted the field taxid with Tom's master key so only who has Tom master and data key should be able to see this",
  "encrypted_field" : tomclientEncryption.encrypt(
    TOM_DATA_KEY,
      "This should be encryped with TOM_DATA_KEY",
      "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic"
   )
})

tim_coll_conn_object.insertOne({
    "name" : "Tim",
    "comment" : "I have encrypted the field taxid with Tim's master key so only who has Tim master and data key should be able to see this",
    "encrypted_field" : timclientEncryption.encrypt(
        TIM_DATA_KEY,
        "This should be encryped with TIM_DATA_KEY",
        "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic"
     )
  })

If I connect with client without providing any master key I get:
```

/Users/ivan.grigolon % mongosh

Enterprise replset [direct: primary]> show dbs

csfle_db    8.19 kB
keyVaultDB  61.4 kB

Enterprise replset [direct: primary]> use csfle_db
Enterprise replset [direct: primary]> show collections
Enterprise replset [direct: primary]> db.csfle_coll.find()
[
  {
    _id: ObjectId("6119a91140bcdb554d3eaa2a"),
    name: 'Tom',
    comment: "I have encrypted the field taxid with Tom's master key so only who has Tom master and data key should be able to see this",
    encrypted_field: Binary(Buffer.from("01aed914ebe71549e2ac1380ec8ba7fc9c022e0699d0f543cddea3f6e2fd3a3a9e5f5ecfce798794dd91d324f3af0b710bbbf968b23c9389e4eeb829776df4d8d744f2039d6f463cea84f89c9aefe5007d5b680d330cfdb3650dc19b85695a20bd201f801b009e80865ce64597f5aac90b29", "hex"), 6)
  },
  {
    _id: ObjectId("6119a91140bcdb554d3eaa2b"),
    name: 'Tim',
    comment: "I have encrypted the field taxid with Tim's master key so only who has Tim master and data key should be able to see this",
    encrypted_field: Binary(Buffer.from("013c6a56085bb44e2a944e3dc7bab6dc23024c8ad912f302d255a3feb628609bae80d64fc2a400fdc344af68c1ac7d0653e7f1f22140272babff5b36d0bd67c33b5ce917a17d5cbfe02b1d464daa2a0580527021d9736270b271c9b69ce6e4d8bc810deeae054e216b43bc3e9d028450201b", "hex"), 6)
  }
]
```


Let's test what I can see:

```
bash-3.2$ mongosh --nodb --shell --eval "var TIM_LOCAL_KEY='$TIM_LOCAL_KEY'; var TOM_LOCAL_KEY='$TOM_LOCAL_KEY'"

> keyDB = "keyVaultDB";
keyVaultDB
> keyCOLL = "keyVaultColl";
keyVaultColl
> keyVaultNamespace = keyDB+"."+keyCOLL
keyVaultDB.keyVaultColl
> var tom_ClientSideFieldLevelEncryptionOptions = {"keyVaultNamespace" : keyVaultNamespace,"kmsProviders" : {"local" : {"key" : BinData(0, TOM_LOCAL_KEY)}}}

> tom_csfleDatabaseConnection = Mongo("mongodb://localhost:27017/?replicaSet=replset",tom_ClientSideFieldLevelEncryptionOptions)
> tom_keyVault = tom_csfleDatabaseConnection.getKeyVault();

> TOM_DATA_KEY = notEncryptedConnection.getDB(keyDB).getCollection(keyCOLL).findOne({ keyAltNames: { $in: ["tomKey"] } })._id
> notEncryptedConnection = Mongo("mongodb://localhost:27017/?replicaSet=replset")
> TOM_DATA_KEY = notEncryptedConnection.getDB(keyDB).getCollection(keyCOLL).findOne({ keyAltNames: { $in: ["tomKey"] } })._id
> tomEncryptedClient = Mongo("mongodb://localhost:27017/?replicaSet=replset",tom_ClientSideFieldLevelEncryptionOptions)

> tomkeyVault.getKeyByAltName("tomKey")
ReferenceError: tomkeyVault is not defined

> tomkeyVault = tomEncryptedClient.getKeyVault()
KeyVault class for mongodb://localhost:27017/?replicaSet=replset&serverSelectionTimeoutMS=2000

> tomkeyVault.getKeyByAltName("tomKey")

> tom_db_conn_object = tomEncryptedClient.getDB("csfle_db");

> tom_coll_conn_object = tom_db_conn_object.getCollection("csfle_coll");

> tom_coll_conn_object.find()

MongoCryptError: HMAC validation failure
> tom_coll_conn_object.findOne()
{
  _id: ObjectId("6119a91140bcdb554d3eaa2a"),
  name: 'Tom',
  comment: "I have encrypted the field taxid with Tom's master key so only who has Tom master and data key should be able to see this",
  encrypted_field: 'This should be encryped with TOM_DATA_KEY'
}

> tom_coll_conn_object.findOne({name:'Tim'})
MongoCryptError: HMAC validation failure
> tom_coll_conn_object.insertOne({
...   "name" : "Tom",
...   "comment" : "I have encrypted the field taxid with Tom's master key so only who has Tom master and data key should be able to see this",
...   "encrypted_field" : tomclientEncryption.encrypt(
.....     TOM_DATA_KEY,
.....       "This should be encryped with TOM_DATA_KEY",
.....       "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic"
.....    )
... })
ReferenceError: tomclientEncryption is not defined
> tomclientEncryption = tomEncryptedClient.getClientEncryption()
ClientEncryption class for mongodb://localhost:27017/?replicaSet=replset&serverSelectionTimeoutMS=2000
>

> tom_coll_conn_object.insertOne({
...   "name" : "Tom",
...   "comment" : "I have encrypted the field taxid with Tom's master key so only who has Tom master and data key should be able to see this",
...   "encrypted_field" : tomclientEncryption.encrypt(
.....     TOM_DATA_KEY,
.....       "This should be encryped with TOM_DATA_KEY",
.....       "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic"
.....    )
... })
{
  acknowledged: true,
  insertedId: ObjectId("6119ab3e8d213dbdac950bba")
}
> tom_coll_conn_object.insertOne({
...   "name" : "Tom",
...   "comment" : "I have encrypted the field taxid with Tom's master key so only who has Tom master and data key should be able to see this",
...   "encrypted_field" : tomclientEncryption.encrypt(
.....     TOM_DATA_KEY,
.....       "This should be encryped with TOM_DATA_KEY",
.....       "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic"
.....    )
... })
{
  acknowledged: true,
  insertedId: ObjectId("6119ab3f8d213dbdac950bbb")
}
> tom_coll_conn_object.find()
MongoCryptError: HMAC validation failure
> tom_coll_conn_object.find({name:'Tom'})
[
  {
    _id: ObjectId("6119a91140bcdb554d3eaa2a"),
    name: 'Tom',
    comment: "I have encrypted the field taxid with Tom's master key so only who has Tom master and data key should be able to see this",
    encrypted_field: 'This should be encryped with TOM_DATA_KEY'
  },
  {
    _id: ObjectId("6119ab3e8d213dbdac950bba"),
    name: 'Tom',
    comment: "I have encrypted the field taxid with Tom's master key so only who has Tom master and data key should be able to see this",
    encrypted_field: 'This should be encryped with TOM_DATA_KEY'
  },
  {
    _id: ObjectId("6119ab3f8d213dbdac950bbb"),
    name: 'Tom',
    comment: "I have encrypted the field taxid with Tom's master key so only who has Tom master and data key should be able to see this",
    encrypted_field: 'This should be encryped with TOM_DATA_KEY'
  }
]
>
```