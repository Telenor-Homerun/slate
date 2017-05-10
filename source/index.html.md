---
title: Tikky Partner API Reference

language_tabs:
  - code

toc_footers:
  - <a href='https://tikky.no'>Tikky.no</a>
  - <a href='https://tikky.se'>Tikky.se</a>

includes:
  - errors

search: true
---

# Introduction

Tikky is a system that enables in-home deliveries of goods and services.

This is accomplished by creating time-limited digital keys, which are forwarded to digital locks installed in end-users’ homes. The digital keys can be granted to merchants and service providers (partners).

The Partner REST API allows partners to create digital keys (accesses) programmatically. Typically, a partner will use the Partner REST API to create an access as part of the end-user checkout process, if the user has selected “Tikky in-home delivery” as a delivery option.

<aside class="success">
To get access to the Partner API, please contact us at august@tikky.io.
</aside>

## Integration

During the checkout stage, the merchant should use the Partner API to check whether the current user is registered with Tikky. If so, then the user should be presented with the Tikky in-home delivery option, for example, as a checkbox on the checkout page.

If the user selects the in-home delivery option, then the merchant should use the Partner API to create an access. The access returned does not yet have a pincode. It can take a few seconds before the pincode is successfully forwarded to the digital door lock.

After route planning, and normally at the day of delivery, it’s time to use the Partner API to fetch the created pincode. After the pincode has been used, and the delivery is completed, the Partner API should be used to mark the access as completed. At this point, the pincode will be deleted from the door lock.

## Invoking the API

Request data must be supplied as a JSON object in the request body. Response data will be returned as a JSON object in the response body.

Timestamps should be in UTC in ISO-8601 format. For example: `2017-04-
26T13:21:54Z`.

## RSA Public Keys

```shell
openssl genrsa -out xyz.pem 2048
openssl rsa -pubout -in xyz.pem -out xyz.pub
````
> Replace ```xyz``` with a suitable partner name.

In order to use the Partner REST API, the partner must create an RSA key pair. Send the public key (xyz.pub) to us, and we will associate it with your partner account.

Digital keys created on your behalf will be encrypted with your public key. Typically, this encryption is done by the lock itself, so the digital key will not be usable by anyone else than you. You use your private key to decrypt the digital key.

# Authentication

> To request an authentication token, issue the following request:

```shell
curl -X POST https://prod.tikky.io/api/partners/v1/sessions -H 'accept: application/json' -H 'content-type: application/json' -d '{ "email": "partner@partner.xyz", "key": "secretpassword" }'
```

> JSON response:

```shell
{ "token": "EPLzISLquKJtAqH3fI82WIVTcBQzOjtYgbftRL2N2Vw" }
````

Operations requires a valid authentication token. The token can be cached by the client, and is long-lived (it survives up to 30 days of inactivity). However, the client should be prepared to re-create the token if a 401 Unauthorized is received.

The authentication token must be used in the HTTP Authorization header on subsequent calls:

`Authorization: Bearer <token>`

### Request

`POST https://prod.tikky.io/api/partners/v1/sessions`

Parameter | Description
--------- | -----------
email | The partner email address.
key | The partner secret key

# Users
Tikky users are explicitly associated with Tikky partners. A user may be registered with Tikky with their work email address, but may be associated with you as a partner by any other email address.

## Get All Users

```shell
curl -X GET "https://prod.tikky.io/api/partners/v1/users" -H "accept: application/json" -H "Authorization: Bearer EPLzISLquKJtAqH3fI82WIVTcBQzOjtYgbftRL2N2Vw"
```

Returns all users associated with the partner.

### Request

`POST https://prod.tikky.io/api/partners/v1/users`

## Get One User
> Get a user by the account name that you as a partner identify the user (typically, by email address).

```shell
curl -X GET "https://prod.tikky.io/api/partners/v1/users/foodshopper@foo.com" -H "accept: application/json" -H "Authorization: Bearer EPLzISLquKJtAqH3fI82WIVTcBQzOjtYgbftRL2N2Vw"
```
> JSON Response:

```shell
{
  "email": "foodshopper@foo.com",
  "locations": [
    {
      "id": "9af4d7b5-f347-40c1-9715-96e64e534cb2", "name": "Home",
      "street": "Parkveien 63",
      "postalCode": "NO-0270",
      "city": "Oslo",
      "country": "no",
      "locks": [
        {
        "id": "efaeb306-cf5a-4ee4-b553-ec732a1b04a0", "name": "Front Door",
        "type": "Yale Doorman"
        }
      ]
    }
  ]
}
```
The returned user object contains an array of locations, and each location contains an array of locks. Most users have one location and one lock.

### Request

`POST https://prod.tikky.io/api/partners/v1/users/<account>`

<aside class="notice">
The call returns 404 Not Found if the user is not explicitly associated with the partner.
</aside>

# Accesses
An access is defined as a time-limited digital key assigned to a given lock.

## Create Access

```shell
curl -X POST "https://prod.tikky.io/api/partners/v1/accesses" -H "accept: application/json" -H "Authorization: Bearer EPLzISLquKJtAqH3fI82WIVTcBQpOjtYgbftRL2N2Vw" -H "content-type: application/json" -d "{ \"lockId\": \"efaeb306-cf5a-4ee4-b553-ec732a1b04a0\", \"startDate\": \"2017-04-27T06:00:00Z\", \"endDate\": \"2017-04-27T14:00:00Z\", \"orderNumber\": \"order-00001\"}"
````

The Access object grants a partner access to a lock identified by its `lockId`.

The Access object will eventually have an associated digital key. The format of the digital key is dependent on the type of lock. For example, for a Yale Doorman lock, the digital key is a 6-digit pincode. The digital key will only be valid from `startDate` to `endDate`.

The `startDate` and `endDate` fields must follow these rules:

* The start date must be before the end date.
* The start date must be at least one hour from now.
* The end date must be at least one hour from the start date.
* The end date can't be more than 24 hours after the start date.
* Neither date can be more than one month away.

The digital key can be fetched via `/accesses/{id}/key`.

A partner can not have overlapping Access for the same `lockId`. That is, for a given lock, only one Access is allowed within the time window specified by `startDate` and `endDate`. Partners should therefore keep the time window for accesses as small as possible.

### Request

`POST https://prod.tikky.io/api/partners/v1/accesses`

Parameter | Required | Description
--------- | -------- | -----------
lockId | yes | The ID of the lock used by this access.
startDate | yes | When the key should begin to be valid (ISO-8601 datetime).
endDate | yes | When the key should stop being valid (ISO-8601 datetime).
orderNumber | no | The partner's order ID. The order number is visible to the user in the Tikky app.
productName | no | Name of the product or service.
productImageUrl | no | URL to an image of the product or service. Will be displayed to the user in the Tikky app.
personName | no | Name of person delivering the product or service. May be displayed to the user in the Tikky app.
personImageUrl | no | URL to an avatar image of the person delivering or providing the product or service. May be displayed to the user in the Tikky app.

## Fetch Access
A partner can fetch an access by its ID. 

### Request

`GET https://prod.tikky.io/api/partners/v1/accesses/<accessId>`

## Fetch Digital Key 

> Get the encrypted digital key for a given access:

```shell
curl -X GET "https://prod.tikky.io/api/partners/v1/accesses/8d0d2870-794a-4c32-8145-c62cc432639a/key" -H "accept: application/json" -H "Authorization: Bearer EPLzISLquKJtAqH3fI82WIVTcBQpOjtYgbftRL2N2Vw"
```
> JSON Response:

```shell
{
"key": "lA+HLwSMyGwAwHGi1Q473aW/mjIn85gZxl77pMY3/Fachq5zHS03edS8lbwb7hRyYtwjZPcbvf4Gl/9KnqA38BBiF UCl5FeJjaY291g0pYI9xW5/MSkx6Z+bZxjGyv1FV13Q0Y/5dJYS2nqhp3gcKuiofUNEl6kvh3D4qY0rH7MMbQJK31s 2mkuHckujCXcSrPxHOZ2RMFyz2E1C7RTRW9gBd9HmUHO7XlR5z+SBbmd8LZ0oymGHWd24NK13glP9SpY2IKvvlkmmW kJAdJuE1kfoQZNNOyAyH2+ltjfmGf7ZgS4wvDxdWVd6z6PsJS/pAjW4B5HZWyzaxRzBamZAIQ==",
"format": "encrypted" 
}
````

After the digital key has been successfully forwarded to the lock, it will be encrypted with the partner’s public key on the edge of the network (for example, in the lock itself, or in a gateway connected to the lock). As such, the digital key can not be decrypted by Tikky; only the partner can decrypt the digital key.

The get key request returns the encrypted key (base64 encoded), and must be decrypted with the partner’s private key.

The encrypted digital key (a pincode) in the example to the right is encrypted with the mock partner Acme’s public key. The pincode should be decrypted with the partner’s private key by first decoding the base64 and then apply RSA decrypt with OAEP padding. In this case the decrypted pincode is 873647.

<aside class="notice">
The digital key will not immediately be available on the Access object. This is because the digital key is dispatched to the actual lock, and this can take everything from a few seconds to several minutes. If the digital key is not yet ready, then HTTP status 403 is returned.
</aside>

<aside class="notice">
The access may have been revoked by the end user (via the Tikky app). In that case, HTTP status code 410 is returned.
</aside>

> Decrypting an encrypted key:

```shell
echo lA+HLwSMyGwAwHGi1Q473aW/mjIn85gZxl77pMY3/Fachq5zHS03edS8lbwb7hRyYtwjZPcbvf4Gl/9KnqA38BBiFU Cl5FeJjaY291g0pYI9xW5/MSkx6Z+bZxjGyv1FV13Q0Y/5dJYS2nqhp3gcKuiofUNEl6kvh3D4qY0rH7MMbQJK31s2 mkuHckujCXcSrPxHOZ2RMFyz2E1C7RTRW9gBd9HmUHO7XlR5z+SBbmd8LZ0oymGHWd24NK13glP9SpY2IKvvlkmmWk JAdJuE1kfoQZNNOyAyH2+ltjfmGf7ZgS4wvDxdWVd6z6PsJS/pAjW4B5HZWyzaxRzBamZAIQ== \
base64 -D | openssl rsautl -oaep -decrypt -inkey acme.pem
```

> To decrypt the key using Node's Crypto module:

```js
crypto.privateDecrypt({ key: privateKey,
  padding: constants.RSA_PKCS1_OAEP_PADDING
}, new Buffer(encryptedPincode, 'base64')).toString('utf-8');
````

### Request

`GET https://prod.tikky.io/api/partners/v1/accesses/<accessId>/key`

## Mark delivery as completed
```shell
curl -X PUT "https://prod.tikky.io/api/partners/v1/accesses/8d0d2870-794a-4c32-8145-c62cc432639a/completed" -H "accept: application/json" -H "Authorization: Bearer EPLzISLquKJtAqH3fI82WIVTcBQpOjtYgbftRL2N2Vw"
```
After the delivery has been completed the partner should invoke the access complete call. This causes the digital key to be immediately deleted from the lock.

### Request

`PUT https://prod.tikky.io/api/partners/v1/accesses/<accessId>/completed`

