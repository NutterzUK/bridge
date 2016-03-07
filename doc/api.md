Interface: REST API
===================

MetaDisk exposes a RESTful API for managing buckets and pushing/pulling files
to/from the network. These operations are documented here.

Authentication
--------------

MetaDisk understands 3 methods for authentication, which can vary depending on
the operation you wish to perform.

### HTTP Basic

HTTP Basic authentication is supported as a fallback for situations where key
management is not available or specifically for use before a public key has
been associated with your user account.

Your `username` should simply be your registered email address.

Your `password` should be the SHA-256 hash of your password.

### ECDSA Signatures

Once you have added a public ECDSA key to your account, you can use your
private key to sign messages to the server to avoid sending your credentials in
every request. This is the recommended authentication method.

The string that you are expected to sign differs depending on the type of
request. For POST, PUT, and PATCH requests, you must sign the JSON encoded body
of the request. For GET, DELETE, and OPTIONS requests, you must sign the raw
query string parameters.

In addition to the parameters required for each individual request, you must
also include a `__nonce` parameter. This value should be an integer and must be
incremented with every request. A common practice is to simply use the current
UNIX timestamp.

In addition to the request parameters and nonce, you will also sign the HTTP
method and request path. Ultimately the string you will sign will be:

```
<METHOD>\n<PATH>\n<PARAMS>
```

For example, to generate a signature for creating a new storage bucket, you
will sign:

```
POST\n/buckets\n{"storage":10,"transfer":30,"name":"MyBucket","__nonce":1453222669376}
```

This tells the server that at Tue Jan 19 2016 11:57:49 GMT-0500 (EST), you
authorized a request to created a bucket called "MyBucket" with 10GB of capacity
and 30GB of transfer. This request cannot be replayed, since the nonce cannot
be reused.

Once you have generated the signature, it must be encoded as HEX and included
in the `x-signature` header.

In addition you must supply the public key for verifying the signature in the
`x-pubkey` header.

MetaDisk will first lookup the user account to which the supplied public key is
registered and then use it to verify the signature.

### Single Use Tokens

There are 2 cases where signing the request body is not efficient for the client
and verifying the signature is not efficient for the server. This is when the
user wishes to push or pull data to or from a bucket.

If a file is quite large, the server would have to buffer it's entire contents
in order to verify the signature. This is where single-use tokens come into
play.

Instead of signing the upload request, you sign a request for a `token` that
corresponds to a bucket, then use that token in the `x-token` header in your
upload request. This also provides a way for users to grant access to others to
upload files to their bucket by generating a token for them (since anyone with
the token may upload the file to the bucket).

Tokens can be configured to expire (default is 5 minutes) and may only be used
once.

Public Endpoints
----------------

### Create User

* **Method:** `POST`
* **Path:** `/users`
* **Parameters:**
  * `email` - (String) valid email address
  * `password` - (String) SHA-256 hash of password

### Activate User

* **Method:** `GET`
* **Path:** `/activations/:token`
* **Parameters:**
  * `redirect` - (String) redirect url - *optional*

Private Endpoints
-----------------

### Register Public Key

* **Method:** `POST`
* **Path:** `/keys`
* **Parameters:**
  * `key` - (String) hex encoded ECDSA public key

### Fetch Public Keys

* **Method:** `GET`
* **Path:** `/keys`
* **Parameters:** *none*

### Revoke Public Key

* **Method:** `DELETE`
* **Path:** `/keys/:pubkey`
* **Parameters:** *none*

### Fetch All Buckets

* **Method:** `GET`
* **Path:** `/buckets`
* **Parameters:** *none*

### Get Bucket By ID

* **Method:** `GET`
* **Path:** `/buckets/:id`
* **Parameters:** *none*

### Create New Bucket

* **Method:** `POST`
* **Path:** `/buckets`
* **Parameters:**
  * `storage` - (Number) storage space in GB
  * `transfer` - (Number) transfer bandwidth in GB
  * `name` - (String) human readable bucket name
  * `pubkeys` - (Array) list of public key IDs; optional

### Destroy Bucket

* **Method:** `DELETE`
* **Path:** `/buckets/:id`
* **Parameters:** *none*

### Update Bucket By ID

* **Method:** `PATCH`
* **Path:** `/buckets/:id`
* **Parameters:**
  * `storage` - (Number) storage space in GB; optional
  * `transfer` - (Number) transfer bandwidth in GB; optional
  * `name` - (String) human readable bucket name; optional
  * `pubkeys` - (Array) list of public key IDs; optional

### Create Bucket Token

* **Method:** `POST`
* **Path:** `/buckets/:id/tokens`
* **Parameters:**
  * `operation` - (String) one of "PUSH" or "PULL"

### Store File In Bucket

* **Method:** `PUT`
* **Path:** `/buckets/:id`
* **Parameters:** *none*

> Requires `x-token` header. Payload should be the raw file stream.

### Fetch File From Bucket

* **Method:** `GET`
* **Path:** `/buckets/:id/:filehash`
* **Parameters:** *none*

> Requires `x-token` header.