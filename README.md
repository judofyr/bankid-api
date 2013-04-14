# BankID API documentation

BankID is a standard authorization protocol used in Norway. It allows users to
use the same password and two-factor authentication to log in to different
merchants.

This is an attempt at documenting the (proprietary) protocol.

## General overview

BankID is used when a web site (**merchant**) needs to securely identify its
**users**. Users will have to enter their social security number
("fødselsnummer"), a password, and a one-time password (either generated on a
custom device or sent to their phone number). A **central server** handles the
actual authentication, allowing the user to use the same authentication method
across distinct merchants.

A BankID **client** (e.g. the Java applet) communicates with the merchant and
the central server over HTTPS. Requests are sent as HTTP POST using standard
form-data encoding. Binary data is encoded using base64. In addtion to HTTPS,
every request is encrypted using AES-256-CBC and RSA-2048. The client
includes a set of known public keys belonging to the central server; the
merchant's public key is requested from the central server.

The Java applet is instantiated with **parameters** generated by the merchant.
These parameters includes (but is not limited to): URLs to the central- and merchant's
server, valid IPs for domains (to prevent MITM attacks), timeout variables,
language settings, authentication type. The parameters are signed by the
merchant. The client itself doesn't validate the signature, but rather just
passes it on to the central server.

Every HTTP request consists of two "levels": an envelope that contains a RPC
request. Example:

```ruby
# GetMerchantKey-request:
req = {
  "bh" => "GetMerchantKey",
  "ao" => "java",
  "tf" => "5.3.2",
  "xx" => "3.7",
  "ij" => "...",
  "dg" => "...",
  "df" => "...",
}

# Encode as form-data
data = encode(req)

# Envelope
env = {
  "ao" => "java",
  "tf" => "5.3.2",
  "bu" => "3.7",
  "bt" => "...",
  "edo" => base64(encrypt_data(data)),
  "eko" => base64(encrypt_key(key)),
  "eao" => base64(checksum(key, data)),
}

# This is sent as HTTP POST
body = encode(env)
```

Yes, the protocol full of 2-letter key names. And the keys are different when
talking to the merchant's server (as opposed to the central server).

## Crypto

For every request there's generated a new **secret key** (32 bytes). This
secret key is never directly used, but derivations are created using
HMAC-SHA256.

Given `HMAC_SHA256(key, message)`, the derived keys can be computed as:

```
request_encryption_key =       HMAC_SHA256(secret_key, 'requestEncryptionKey')
request_iv = first 16 bytes of HMAC_SHA256(secret_key, 'requestInitVector')
request_auth_key =             HMAC_SHA256(secret_key, 'requestAuthenticationKey')

response_encryption_key =       HMAC_SHA256(secret_key, 'responseEncryptionKey')
response_iv = first 16 bytes of HMAC_SHA256(secret_key, 'responseInitVector')
response_auth_key =             HMAC_SHA256(secret_key, 'responseAuthenticationKey')
```

### Public keys

The Java applet includes five pairs of keys (included in this repo as
`keys/1a.pub` to `keys/5b.pub`). I haven't quite figured out what the different
keys mean.

### Secret key encryption

The secret key is encrypted using the server's public key (RSA). At the moment
it seems that it uses `keys/5a.pub`.

### Data encryption

General data is encrypted using AES-256-CBC. See the derived keys above for the
key and initialization vector.

### Signatures

Although BankID uses HMAC-SHA256 for key derivation, it uses HMAC-SHA1 for
signing data. Yes, you heard that right: When it wants to sign something it
derives the signing/authentication key from the secret key using HMAC-SHA256,
then it uses that to sign the data using HMAC-SHA1.

Let me repeat it in code:

```
# As seen above:
request_auth_key = HMAC_SHA256(secret_key, 'requestAuthenticationKey')
# Signing
signature = HMAC_SHA1(request_auth_key, data)
```

## Central communication

The URL for the central server can be found as `serverURL` in the *parameters*.
At the moment it seems to always be `https://activation1.bankid.no/CentralServerFE/Gateway`.

Requests are POSTed to this URL using the following envelope.

### Envelope

Request:

| Name | Description                                            | Example                        |
| ---  | ---                                                    | ---                            |
| bu   | Constant                                               | `3.7`                          |
| tf   | Constant                                               | `5.3.2`                        |
| ao   | Constant                                               | `java`                         |
| bt   | `tid` in *parameters*                                  | `qG7GN5MYA/D4NjzNrvy1KoBgxfA=` |
| edo  | Encrypted request data (base64)                        | ...                            |
| eko  | Encrypted request key (base64)                         | ...                            |
| eao  | Signature of `encrypted_data + encrypted_key` (base64) | ...                            |

Response:

| Name      | Description                            | Example |
| ---       | ---                                    | ---     |
| ErrorCode | Only present if there's an error       | `1412`  |
| edo       | Encrypted response data (base64)       | ...     |
| eao       | Signature of `encrypted_data` (base64) | ...     |

### GetMerchantKey

Request:

| Name | Description                 | Example                   |
| ---  | ---                         | ---                       |
| xx   | Constant                    | `3.7`                     |
| tf   | Constant                    | `5.3.2`                   |
| ao   | Constant                    | `java`                    |
| bh   | Constant                    | `GetMerchantKey`          |
| ij   | Metadata? Can be left empty | ``                        |
| dg   | Parameter signature         | `aW10hQ5GQhS9NwHjE` ...   |
| df   | Parameters (form-data)      | `type=application%2F` ... |

The parameter signature (dg) is available as `popSignature` in
the *parameters*. `popParameters` contains a comma-separated,
base64-encoded, list of parameter keys. You must create a form-data
string containing these parameters (df).

Response:

| Name | Description                        | Example       |
| ---  | ---                                | ---           |
| bn   | Error code. `0` = success          | `0`           |
| dj   | Your IP-address                    | `80.80.80.80` |
| re   | Some sort of regexp blacklist      |               |
| ab   | Some sort of JavaScript? blacklist |               |
| dm   | X509-encoded certificate (base64)  |               |

The certificate contains the public key to the merchant's server.

### GetUserInfo

Request:

| Name | Description           | Example       |
| ---  | ---                   | ---           |
| xx   | Constant              | `3.7`         |
| tf   | Constant              | `5.3.2`       |
| ao   | Constant              | `java`        |
| bh   | Constant              | `GetUserInfo` |
| bq   | User ID (SSN)         |               |
| ar   | Merchant name         | `DNB Bank`    |
| cg   | Either `PER` or `EMP` | `PER`         |
| cf   | Constant              | `NC`          |

Note that merchant name (`ar`) is the common name extracted from the
certificate from GetMerchantKey.

Response:

| Name  | Description                            | Example |
| ---   | ---                                    | ---     |
| bn    | Error code. `0` = success              | `0`     |
| au    | Number of OTP services                 | `1`     |
| be1 - | Form-data encoded list of OTP services |         |

GetUserInfo returns a list of OTP services. `au` contains the size of the list
and each element is stored as `be1`, `be2` ... `beX` (where `X = au - 1`).

Each element is form-data encoded:

OTP service:

| Name | Description    | Example               |
| ---  | ---            | ---                   |
| bd   | Name           | `Postbanken Digipass` |
| br   | Issuer?        | `DNB`                 |
| bc   | Unknown        | `0`                   |
| bs   | `PER` or `EMP` | `PER`                 |

### OTPAuthenticate

Request:

| Name | Description                                | Example               |
| ---  | ---                                        | ---                   |
| xx   | Constant                                   | `3.7`                 |
| tf   | Constant                                   | `5.3.2`               |
| ao   | Constant                                   | `java`                |
| bh   | Constant                                   | `OTPAuthenticate`     |
| cg   | `PER` or `EMP`                             | `PER`                 |
| bl   | Type (`AUTHENTICATION` or `SIGNING`)       | `AUTHENTICATION`      |
| aq   | Locale                                     | `nb`                  |
| bf   | One-time password                          | `123456`              |
| bd   | OTP name                                   | `Postbanken Digipass` |
| by   | `h` of `bssChannel` from initAuth (base64) |                       |
| ij   | Metadata                                   |                       |
| dc   | Hardware data                              | `dd=...&de=...`       |
| bq   | User ID (SSN)                              |                       |

Hardware data: `de` contains the MAC-address. `dd` is a unique identifier that
is persisted on your computer.

Response:

| Name  | Description               | Example |
| ---   | ---                       | ---     |
| bn    | Error code. `0` = success | `0`     |
| ba    | Data                      |         |
| bj    | Signature                 |         |
| as    | Size of list              | `1`     |
| ad1 - | Form-data encoded list    |         |

`ad` contains elements with the follow structure:

| Name | Description                              | Example                                   |
| ---  | ---                                      | ---                                       |
| ac   | BankID friendly name                     | `Postbanken-BankID`                       |
| ay   | OrderIDRandom                            | `PCemJPUZXFMAHLYPkXorst9MBcg=`            |
| ai   | Force USP Change flag (`N` or `Y`)       | `N`                                       |
| at   | Force USP Change Renew flag (`N` or `Y`) | `N`                                       |
| ap   | Last login timestamp                     | `20130409185726`                          |
| am   | First login timestamp                    | `20101015160936`                          |
| dl   | Merchant name                            | `DNB Bank`                                |
| dk   | Merchant title                           | `Postbanken - en del av DnB NOR Bank ASA` |
| dp   | EndUserCertificate (X509, base64)        |                                           |

### Sign

Request:

| Name | Description                           | Example          |
| ---  | ---                                   | ---              |
| xx   | Constant                              | `3.7`            |
| tf   | Constant                              | `5.3.2`          |
| ao   | Constant                              | `java`           |
| bh   | Constant                              | `Sign`           |
| ij   | Metadata                              |                  |
| bl   | `AUTHENTICATION` or `SIGNING`         | `AUTHENTICATION` |
| bp   | SHA-1 of `serverChallenge` (base64)   |                  |
| na   | SHA-256 of `serverChallenge` (base64) |                  |
| bw   | SHA-1 of `clientChallenge` (base64)   |                  |
| ni   | SHA-256 of `clientChallenge` (base64) |                  |
| bk   | `pkcs7` from initAuth                 |                  |
| bv   | `bssChannel` from initAuth            |                  |
| bx   | `bssChannelSignature` from initAuth   |                  |
| bj   | `bj`  from OTPAuthenticate            |                  |
| ba   | `ba` from OTPAuthenticate             |                  |
| ay   | `ay` from OTPAuthenticate             |                  |
| ak   | SHA-1 of `password + userID`          |                  |

## Merchant communication

The URL for the merchant server can be found as `URL` in the *parameters*. You
must first call `GetMerchantKey` to the central server to extract the
merchant's public key.

Requests are POSTed to this URL using the following envelope.

### Envelope

Request:

| Name      | Description                                          | Example    |
| ---       | ---                                                  | ---        |
| operation |                                                      | `initAuth` |
| encData   | Encrypted request data (base64)                      | ...        |
| encKey    | Encrypted request key (base64)                       | ...        |
| encAuth   | Signature of `operation + encData + encKey` (base64) | ...        |

Note that (as opposed to the central envelope) the encrypted data and key is
encoded in base64 when the signature is computed.

Response:

| Name    | Description                      | Example |
| ---     | ---                              | ---     |
| encData | Encrypted response data (base64) | ...     |
| encAuth | Signature of `encData` (base64)  | ...     |

### initAuth

Request:

| Name            | Description                 | Example                        |
| ---             | ---                         | ---                            |
| operation       | Name of the operation       | `initAuth`                     |
| clientChallenge | 20 random bytes (base64)    | `hvvsjVgvo6ULUSW3ic2DDBPZ0uk=` |
| carrier         | Constant                    | `NC`                           |
| tokenType       | `token` from the parameters | `roaming`                      |
| clientIp        | IP address of the client    | `80.80.80.80`                  |

Response:

| Name                | Description              | Example  |
| ---                 | ---                      | ---      |
| userID              | SSN (YYYYMMDDXXXXX)      | You wish |
| serverChallenge     | ??                       |          |
| pkcs7               | Some kind of certificate |          |
| bssChannel          | base64 encoded data      |          |
| bssChannelSignature | Some kind of signature   |          |

### verifyAuth

Request:

| Name      | Description           | Example    |
| ---       | ---                   | ---        |
| operation | Name of the operation | `initAuth` |
| carrier   | Constant              | `NC`       |
| pkcs7     | `pkcs7` from initAuth |            |
| traceId   | A trace ID?           |            |

