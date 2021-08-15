## Study for CSFLE

### Table of Contents
**[How to git](#how-to-git)**<br>
**[Aws usefull command](#aws-usefull-command)**<br>


## How to git reinder as I don't use it often

## Basic sequence to follow
1. `git pull` - this will download the latest version. If you don't do this and you try to push something you will see something like ! [rejected]        main -> main (fetch first)
2. do your changes locally
3. `git add .` - this is to add the modification to all files, or you can specify the files you want to commit
4. git commit -m "Some commit comment" - this will commit on the local repository
5. git push -  this will commit to the remote repo
6. From vscode you can do this https://marketplace.visualstudio.com/items?itemName=ivangabriele.vscode-git-add-and-commit
6.1. `Cmd + Shift + A (Mac)` and enter a comment
6.2. `Cmd + Shift + X (Mac)` and it is pushed


## Key types

### Master Key
1. This the key that is used to encrypt and decrypt the data key.
2. It is never sent back and forth. It never leave the place were it is stored
3. It can be stored in a KMS or can be LOCAL (not recommended).
4. When it is stored in the KMS, the data-key are sent back and forth to the KMS service for encryption and decryption

### Data key


## Example of manual encryption by using locally managed key

```
bash # Using bash
# Create the master-key
echo "export DEV_LOCAL_KEY=\"$(head -c 96 /dev/urandom | base64 | tr -d '\n')\"" >> ~/.profile
```

# This is not immediately availabe unless you create a new shell or source the ~/.profile
```
echo $DEV_LOCAL_KEY  
source ~/.profile
echo $DEV_LOCAL_KEY
```

Connect to the cluster:

```
mongosh --eval "var LOCAL_KEY = '$DEV_LOCAL_KEY' " --shell --nodb
...
> LOCAL_KEY
r9567Ko0B5On9jmkv1fQj0v5vzQKjbGODn7uVZXe4XtPYawa6Y+43kZ5E+JTPrPJsyamrn1CXlOG16hQ4vsmCqItY+LOViF/wET475r47Kr4iAGz12zVrLmw/p1N/e5B
```

## Create the encryption configuration
```
var ClientSideFieldLevelEncryptionOptions = {
  "keyVaultNamespace" : "ivanEncryptDB.ivanEncryptColl",
  "kmsProviders" : {
    "local" : {
      "key" : BinData(0, LOCAL_KEY)
    }
  }
}

## Output is:
{
  keyVaultNamespace: 'ivanEncryptDB.ivanEncryptColl',
  kmsProviders: {
    local: {
      key: Binary(Buffer.from("afde7aecaa340793a7f639a4bf57d08f4bf9bf340a8db18e0e7eee5595dee17b4f61ac1ae98fb8de467913e2533eb3c9b326a6ae7d425e5386d7a850e2fb260aa22d63e2ce56217fc044f8ef9af8ecaaf88801b3d76cd5acb9b0fe9d4dfdee41", "hex"), 0)
    }
  }
}
```

Note that nothing has yet been created on MongoDB side so far:


## Connect with Encryption Support.
In mongosh, use the Mongo() constructor to establish a database connection to the target cluster. Specify the ClientSideFieldLevelEncryptionOptions document as the second parameter to the Mongo() constructor to configure the connection for client-side field level encryption:
Use the csfleDatabaseConnection object to access client-side field level encryption shell methods.
```
csfleDatabaseConnection = Mongo(
  "mongodb://localhost:27017/?replicaSet=replset",
  ClientSideFieldLevelEncryptionOptions
)

And we can inspect this as:
> csfleDatabaseConnection
mongodb://localhost:27017/?replicaSet=replset&serverSelectionTimeoutMS=2000

```


## Create the Key Vault object in order to access the methods to create the keys
```
keyVault = csfleDatabaseConnection.getKeyVault();
KeyVault class for mongodb://localhost:27017/?replicaSet=replset&serverSelectionTimeoutMS=2000
```

## Create the Data Encryption Key.
Note that the  KeyVault.createKey() method on the keyVault object has the format:
```
keyVault.createKey(
  "local",
  [ "keyAlternateName" ]
)
```
Use the "keyAlternateName" to specify a findable key name that you will use to access the data you have encrypted with that specific key

```
> keyVault.createKey(
...   "local")
UUID("969270e0-c546-4a8b-8a22-157126fa571c")
> keyVault.createKey( "local",["ivanKey"])
UUID("6c2e42bd-5054-4419-ab3f-37f7ea75163a")
> keyVault.createKey( "local",["ivanSharedKey"])
UUID("1ea4c446-00ef-4db7-a6e4-aad1d94de3d9")
> keyVault.createKey( "local",["PaulSharedKey"])
UUID("15da4b97-34b2-4d3e-9c4d-e81da50ea694")
```

If you are providing the data encryption key to a client in order to configure automatic client-side field level encryption, you must use the base64 representation of the UUID string.
You can run the following operation in mongosh to convert a UUID hexadecimal string to its base64 representation:
```
UUID("b4b41b33-5c97-412e-a02b-743498346079").base64()
```

Supply the UUID of your own data encryption key to this command, as returned from createKey() above, or as described in Retrieve an Existing Data Encryption Key.


Once you have done this, using another shell I verified what was created on the database:

```
MongoDB Enterprise replset:PRIMARY> use ivanEncryptDB
MongoDB Enterprise replset:PRIMARY> db.ivanEncryptColl.find().pretty()
...
{
	"_id" : UUID("6c2e42bd-5054-4419-ab3f-37f7ea75163a"),
	"keyMaterial" : BinData(0,"pSQw2kQUK+NykXcElBwdz0A6fZQjarWVh3ErCOmxvp+jC40a1371WSaXEBT0VpiH9QBvpFLkc4TuJJn0ZTeRupcXEzyj79xw6Y072E+2N6NJxuinZIYkrTPO0BvUbBsAmmI31OuVC51kwEvAoPTPmQOX6+VZB3utF0l9F+ntWrGLDRccklwMpZPLPrW6+kylilipnEqsnFQ0sNYHfcj3qg=="),
	"creationDate" : ISODate("2021-08-15T00:41:07.142Z"),
	"updateDate" : ISODate("2021-08-15T00:41:07.239Z"),
	"status" : 0,
	"masterKey" : {
		"provider" : "local"
	},
	"keyAltNames" : [
		"ivanKey"
	]
}
...
}
MongoDB Enterprise replset:PRIMARY>

```

I can do the same with the getKeys() method, but have a look at the difference in how the keyMaterial is presented:
```
> keyVault.getKeys()
[
...
  {
    _id: UUID("6c2e42bd-5054-4419-ab3f-37f7ea75163a"),
    keyMaterial: Binary(Buffer.from("a52430da44142be372917704941c1dcf403a7d94236ab59587712b08e9b1be9fa30b8d1ad77ef55926971014f4569887f5006fa452e47384ee2499f4653791ba9717133ca3efdc70e98d3bd84fb637a349c6e8a7648624ad33ced01bd46c1b009a6237d4eb950b9d64c04bc0a0f4cf990397ebe559077bad17497d17e9ed5ab18b0d171c925c0ca593cb3eb5bafa4ca58a58a99c4aac9c5434b0d6077dc8f7aa", "hex"), 0),
    creationDate: ISODate("2021-08-15T00:41:07.142Z"),
    updateDate: ISODate("2021-08-15T00:41:07.239Z"),
    status: 0,
    masterKey: { provider: 'local' },
    keyAltNames: [ 'ivanKey' ]
  },
...
]
```









Do you want to see how the data-key are stored? (Optional Stuff)

```
cluster = Mongo("mongodb://localhost:27017/?replicaSet=replset")
db = cluster.getDB("ivanEncryptDB.ivanEncryptColl");
coll = db.getCollection("ivanEncryptColl");
coll.find()
[
  {
    _id: UUID("af412470-9b64-4159-8d3f-452392e7e6a2"),
    keyMaterial: Binary(Buffer.from("0c26ad7119a8c75e674e05d0edc20d7d120e8732e78fbc3691eea92d2dc7899fd0210a0658ca8dcb6991a809fb89a292b1d407d97180654d1ca18a6a71edc3a34b7acf7464200c964c6cda3c16766fa18b55c31b542f081dca1ee380eb54d090d40a864f56dff93e3cb3988423493cdbeca738bcd73261e92763d4613b6e742d78727ff033ade3c18b6507c192a53b60d3d954c3cda83a10e95cf7c1f4c05ebd", "hex"), 0),
    creationDate: ISODate("2021-08-13T23:48:25.493Z"),
    updateDate: ISODate("2021-08-13T23:48:25.493Z"),
    status: 0,
    masterKey: { provider: 'local' }
  }
]
```


## Now a client connect and want to encrypt the data:

# Connect and get the master key
```
Logout or login to source the key if you cannot see when echo it
source ~/.profile
mongosh --nodb --shell --eval "var LOCAL_KEY='$DEV_LOCAL_KEY'"
```

# Create the client-side FLE object:
```
var ClientSideFieldLevelEncryptionOptions = {
  "keyVaultNamespace" : "ivanEncryptDB.ivanEncryptColl",
  "kmsProviders" : {
    "local" : {
      "key" : BinData(0, LOCAL_KEY)
    }
  }
}
```

You should see:
```
{
  keyVaultNamespace: 'ivanEncryptDB.ivanEncryptColl',
  kmsProviders: {
    local: {
      key: Binary(Buffer.from("afde7aecaa340793a7f639a4bf57d08f4bf9bf340a8db18e0e7eee5595dee17b4f61ac1ae98fb8de467913e2533eb3c9b326a6ae7d425e5386d7a850e2fb260aa22d63e2ce56217fc044f8ef9af8ecaaf88801b3d76cd5acb9b0fe9d4dfdee41", "hex"), 0)
    }
  }
}
```


```
# Use the Mongo() constructor to create a database connection with the client-side field level encryption options. Replace the mongodb://myMongo.example.net URI with the connection string URI of the target cluster.
```
encryptedClient = Mongo(
  "mongodb://localhost:27017/?replicaSet=replset",
  ClientSideFieldLevelEncryptionOptions
)
```

You should see:
```
mongodb://localhost:27017/?replicaSet=replset&serverSelectionTimeoutMS=2000
```

Retrieve the keyVault object and use the KeyVault.getKey() to retrieve a data encryption key using its UUID:
```
keyVault = encryptedClient.getKeyVault()
keyVault.getKey(UUID("6c2e42bd-5054-4419-ab3f-37f7ea75163a"))
```

I get:
```
[
  {
    _id: UUID("6c2e42bd-5054-4419-ab3f-37f7ea75163a"),
    keyMaterial: Binary(Buffer.from("a52430da44142be372917704941c1dcf403a7d94236ab59587712b08e9b1be9fa30b8d1ad77ef55926971014f4569887f5006fa452e47384ee2499f4653791ba9717133ca3efdc70e98d3bd84fb637a349c6e8a7648624ad33ced01bd46c1b009a6237d4eb950b9d64c04bc0a0f4cf990397ebe559077bad17497d17e9ed5ab18b0d171c925c0ca593cb3eb5bafa4ca58a58a99c4aac9c5434b0d6077dc8f7aa", "hex"), 0),
    creationDate: ISODate("2021-08-15T00:41:07.142Z"),
    updateDate: ISODate("2021-08-15T00:41:07.239Z"),
    status: 0,
    masterKey: { provider: 'local' },
    keyAltNames: [ 'ivanKey' ]
  }
]
```

The following operation issued from mongosh explicitly encrypts the taxid field as part of a write operation.

```
#  Get the db and collection objects as needed
db = encryptedClient.getDB("hr");
coll = db.getCollection("employees");
```

#Retrieve the ClientEncryption object and use the ClientEncryption.encrypt() method to encrypt a value using a specific data encryption key UUID and encryption algorithm:
```
replset [primary]> clientEncryption = encryptedClient.getClientEncryption();
ClientEncryption class for mongodb://localhost:27017/?replicaSet=replset&serverSelectionTimeoutMS=2000
```

Insert a document and specifically ask to encrypt the "taxid" field:
```
coll.insertOne({
  "name" : "J. Doe",
  "taxid" : clientEncryption.encrypt(
      UUID("6c2e42bd-5054-4419-ab3f-37f7ea75163a"),
      "123-45-6789",
      "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic"
   )
})
```
Output:
```
{
  acknowledged: true,
  insertedId: ObjectId("61189e344b421ea5f3d62e61")
}

# Let's see how they look
replset [primary]> use hr

replset [primary]> db.employees.find()
[
  {
    _id: ObjectId("61189e344b421ea5f3d62e61"),
    name: 'J. Doe',
    taxid: '123-45-6789'
  }
]
## and if I insert one without encrypted field:

replset [primary]> coll.insertOne({ "name": "Ivan NotSecure", "taxid": "This ID was inserted without encryption" })
{
  acknowledged: true,
  insertedId: ObjectId("61189f0e4b421ea5f3d62e62")
}
replset [primary]> db.employees.find()
[
  {
    _id: ObjectId("61189e344b421ea5f3d62e61"),
    name: 'J. Doe',
    taxid: '123-45-6789'
  },
  {
    _id: ObjectId("61189f0e4b421ea5f3d62e62"),
    name: 'Ivan NotSecure',
    taxid: 'This ID was inserted without encryption'
  }
]
replset [primary]>
```

As you can see from this client I can read both documents because the client has the decryption key. 

But How this look like for clients that do not load the master key:

```
/Users/ivan.grigolon % mongosh
...

Enterprise replset [direct: primary]> use hr

Enterprise replset [direct: primary]> db.employees.find()
[
  {
    _id: ObjectId("61189e344b421ea5f3d62e61"),
    name: 'J. Doe',
    taxid: Binary(Buffer.from("016c2e42bd50544419ab3f37f7ea75163a0296210e592b3c9266151e3136fff133b215ff0e3b9d3e451de8f91aa4be4a026470c34d3cb12965aa58373df2918ea44c50e9710cd8a87ce7cc4ef675258981954cfa2fef059a854f034506bc2123ec48", "hex"), 6)
  },
  {
    _id: ObjectId("61189f0e4b421ea5f3d62e62"),
    name: 'Ivan NotSecure',
    taxid: 'This ID was inserted without encryption'
  }
]
```

What if instead I insert some other names with different keys:
```

```
