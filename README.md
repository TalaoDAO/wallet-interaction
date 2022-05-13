# Interaction between the wallet and Issuers-Verifiers

Updatesd : 13th May 2022  
Author : Thierry Thevenet, thierry.thevenet@talao.io

# Documentation
  
* [Overview](#overview)
  - [Collecting a Verifiable Credential](#collecting-a-verifiable-credential)
  - [Requesting a verifiable presentation](#requesting-a-verifiable-presentation)
  - [Issuer and verifier registry](#issuer-and-verifier-registry) 
  - [Issuer and verifier implementation](#Issuer-and-verifier-implementation)
* [Credential offer protocol](#credential-offer-protocol)
  - [Issuer implementation](#issuer-implementation)
  - [Credential manifest](#credential-manifest)  
* [Presentation request QueryTypes](#presentation-request-query-types) 
  - [Presentation request DIDAuth](#DIDAuth) 
  - [Presentation request QueryByExample](#QueryByExample)
* [Servers return codes](#Servers-return-codes)
* [Universal link](#universal-link)



# Overview

For the interaction of the wallet with Issuers and Verifier, which is a decisive function in the implementation of this type of solution, we have retained the specifications of the W3C Presentation Request as well as the Spruce protocol for the management of the QR Code on a HTTPS transport layer. This protocol has the advantage of being quick and easy to implement while providing most of the necessary functions. There are currently several very promising protocols that are being defined or even in production but these seemed to us too complex and ultimately unnecessary for the current use cases which remain simple.

This protocol is described by Spruce: https://github.com/spruceid/credible#supported-protocols. Talao has enhanced this protocol to adapt it to its own use cases (business logic, governance requirements, etc.). 

This protocol occurs when a user wishes to use his wallet to collect credentials or to present them to access resources. 

In the following document we will noted the options added by Talao as "Talao build".

## Collecting a verifiable credential 

Spruce Credible build  

When the user wants to collect VCs, it is very likely that he will access this service after a first authentication. This wallet protocol does not automatically integrate this first authentication. The user must either authenticate with a pre-existing means of authentication (login/password, openID Connect flow, ...) or possibly use a VC already collected to introduce himself. This process will be used for most of the VCs to be collected such as the identity card, a professional certificate, an electronic bank card, a membership card,…. However, there are special cases where the sender collects information in an anonymous session just before issuing a VC. This is the case of a proof of email, a proof of phone, .... 
 
This VCs collection protocol called Credential Offer is as follows: the issuer displays a QR code (or sends an email with a deep link) which indicates a service URL to which its own DID is added. The wallet requests authorization to the user to access the URL possibly by retrieving information about the issuer from a trusted registry. 

If the user agrees, the issuer transfers a “preview” of the VC with presentation data (color, name, description, etc.) as well as an expiry date of the offer. If the user accepts the proposal, the wallet transfers the DID to the issuer who signs the VC and transfers it to the wallet. 

This protocol is carried by a GET request and a POST request on the URL indicated in the QRCode.

It has been described by Spruce (see below  an extract of the Spruce [README.md]( https://github.com/spruceid/credible#credentialoffer ) ) :



| Wallet                   | <sup>1</sup> |       |                      Server |
| ------------------------ | ------------ | :---: | --------------------------: |
| Scan QRCode <sup>2</sup> |              |       |
| Trust Host               | ○ / ×        |       |                             |
| HTTP GET                 |              |   →   | https://domain.tld/endpoint |
|                          |              |   ←   |             CredentialOffer |
| Preview Credential       |              |       |                             |
| Choose DID               | ○ / ×        |       |                             |
| HTTP POST <sup>3</sup>   |              |   →   | https://domain.tld/endpoint |
|                          |              |   ←   |        VerifiableCredential |
| Verify Credential        |              |       |                             |
| Store Credential         |              |       |                             |

*<sup>1</sup> Whether this action requires user confirmation, exiting the flow
early when the user denies.*  
*<sup>2</sup> The QRCode should contain the HTTP endpoint where the requests
will be made.*  
*<sup>3</sup> The body of the request contains a field `subject_id` set to the
chosen DID.*

![issuer2.png](issuer2.png)


## Requesting a verifiable presentation

 Spruce Credible build  

The presentation of a VC or without any VC can be used for authentication or to request very specific and different services as submit a file, open a bank account, buy online ...

Authentication without VC consists of transferring an empty but signed VP. It is only possible if the verifier already has knowledge of the user's DID, therefore after prior enrollment. 

Authentication with a VC can be used on the first interaction between the user and the verifier. The verifier will be able to specify his request by giving the type of VC expected, or the desired issuer, JSON-LD scheme,.. etc. The verifier will then proceed to check the signatures of the user and the issuers, possibly using a trusted register.  

This protocol called Presentation Request is as follows: the verifier displays a QR code which indicates a service URL to which its own DID is added. The wallet requests authorization to the user to access the URL possibly by retrieving information about the verifier from a trusted registry. Verifying the identity of the verifier is critical to prevent the user from disclosing personal data to untrusted websites.

If the user agrees, the sender transfers his request specifying the type of VC expected. In the case of authentication by DID, the user can accept and send his empty presentation, in the case of a request from particular VCs, the wallet offers the user to select the VCs he wishes to add to his presentation. The list of VCs offered to the user is produced on the basis of the criteria of the verifier's request.

It has been described by Spruce (see below  an extract of the Spruce [README.md]( https://github.com/spruceid/credible#presentationrequest ) ) :

| Wallet                       | <sup>1</sup> |       |                        Server |
| ---------------------------- | ------------ | :---: | ----------------------------: |
| Scan QRCode <sup>2</sup>     |              |       |
| Trust Host                   | ○ / ×        |       |                               |
| HTTP GET                     |              |   →   |   https://domain.tld/endpoint |
|                              |              |   ←   | VerifiablePresentationRequest |
| Preview Presentation         |              |       |                               |
| Choose Verifiable Credential | ○ / ×        |       |                               |
| HTTP POST <sup>3</sup>       |              |   →   |   https://domain.tld/endpoint |
|                              |              |   ←   |                        Result |

*<sup>1</sup> Whether this action requires user confirmation, exiting the flow
early when the user denies.*  
*<sup>2</sup> The QRCode should contain the HTTP endpoint where the requests
will be made.*  
*<sup>3</sup> The body of the request contains a field `presentation` set to the
verifiable presentation.*


![verifier_cross_device.png](verifier_cross_device.png)

## Issuer and verifier registry 

Talao build  

### Motivation
The protocol of interaction between the wallet and an Issuer or a Verifier currently used by Credible is light, simple and quick to implement, However it does not allow the user of the wallet to ensure the identity of the other party but only the domain name specified in the URL encoded in the QR Code. On the other hand, a simple solution based on access to a public register of Issuers / Verifier makes it possible to obtain more information for the user and therefore better control without considerably increasing the complexity of the protocol. However hhis service must be considered as optional due to correlation issues.

### Issuer and verifier implementation
The Issuer (or Verifier) DID is passed as an argument in the QRcode callback URL.

example : https://talao.co/....?issuer=did:ethr:0xee09654eedaa79429f8d216fa51a129db0f72250).


# Credential offer protocol

Talao build  

## Motivation

For holders wishes to engage with Issuers to acquire credentials, there must exist a mechanism for assessing what inputs are required from an issuer to process a request for credential issuance. A manifest is a common data format for describing the inputs a user must provide to an Issuer and the way the VCs should be presented. This draft has been inpired by the Credential Manfifest specification with a very limited implementations :

- display output descriptors as labels (name, description) et templates objetcs (label name, description, color,...),
- input descriptor,
- share link : a way to use the wallet to link to a cloud service (vault, etc),
- challenge to sign a VP.

## Issuer implementation
When the Credible wallet makes a GET to the Issuer endpoint, a JSON is returned to the wallet (Issuer GET response):

```javascript
{
           "type": "CredentialOffer",
           "credentialPreview": {...},
           "expires" : "2022-09-01T19:29:39Z"
 }

```

The modification consists in adding optional attributes to the JSON returned by the Issuer (Issuer GET response).
 
The "id" attribute will be used to follow a wallet session with a static QRcode  
The "shareLink" attribute is an UR to be presented for share link as user convenience.    
The "challenge" and domain arttributes will be used for DID_auth response or self-issued.  
The "credential_manifest" attribute is used to define the expected display options of the VC in the wallet (outpout_descriptors).  

Challenge, id, sharlink and credential manifiest are optional attributes.  

example:

```javascript
{
          "id ": "uuid:urn:...",
           "type": "CredentialOffer",
           "credentialPreview": {...},
           "expires" : 2022-09-01T19:29:39Z",
            "challenge" : "mjh45RT56",
            "domain" : "talao.co",
            "shareLink" : "https://talao.co/credential/link?issuer=did:tz:tz1e5YakmACgZZprF7YWHMqnSvcWVXZ2TsPW&id=urnn:idnn:4564:...",
            "credential_manifest" : {.. credential manifest....}
                       
}
```
The wallet response will be :

```javascript
{
           "id" : "uuid:urn....",
           “Subject_id”, ”did: tz: tz1e5YakmACgZZprF7YWHMqnSvcWVXZ2TsPW”,
            “verifiable_presentation”: {.verfiable presentation as self issued VC..}
}
```

verifiablePresentation is a self issued VC/VP bound with nonce if available in the request. This VP includes all the existing profile attributes if wallet holder consents.  


### Credential manifest

For credential manifest see https://identity.foundation/credential-manifest/ for more information.  

an example here :  


```javascript

{
              "id":"PC_01",
              "issuer":{
                "id":"did:tz:tz1NyjrTUNxDpPaqNZ84ipGELAcTWYg6s5Du",
                "name":"Talao issuer",
                "styles": {
                  "thumbnail": {
                    "uri": "https://talao.mypinata.cloud/ipfs/QmU8z1aDDpbgVjXwU8nMvgU3YD4VeUzHTkitsELZRFBBZa",
                    "alt": "Talao logo"
                  },
                  "background": {
                  "color": "#ffffff"
                  },
                  "text": {
                  "color": "#d4d400"
                  }
                }
              },
              "output_descriptors":[
                        {
                          "id": "PCDS_01",
                          "schema": "PCDSAuditorCertificate",
                          "display": {
                            "title": {
                              "path": ["$.name"],
                              "schema": {
                                "type": "string"
                              },
                              "fallback": "Title fallback"
                            },
                            "subtitle": {
                              "path": ["$.description"],
                              "schema": {
                                "type": "string"
                              },
                              "fallback": "Subtitle fallback"
                            },
                            "description": {
                              "text": "This is an electronic version of a PCDS auditor certificate for testing purposes."
                            },
                            "properties": [
                              {
                                "path": ["$.credentialSubject.category"],
                                "schema": {
                                  "type": "string"
                                },
                                "fallback": "Unknown category",
                                "label": "Certificate category"
                              },
                              {
                                "path": ["$.credentialSubject.familyName"],
                                "schema": {
                                  "type": "string"
                                },
                                "fallback": "Unknown auditor name",
                                "label": "Auditor name"
                              },
                              {
                                "path": ["$.credentialSubject.certificationDate"],
                                "schema": {
                                  "type": "string"
                                },
                                "fallback": "Unknown certification date",
                                "label": "Certification Date"
                              },
                              {
                                "path": ["$.credentialSubject.validity"],
                                "schema": {
                                  "type": "string"
                                },
                                "fallback": "Unknown validity",
                                "label": "Certificate validity"
                              }
                            ]
                          }
                        }
              ],
              "presentation_definition":{}
            }
```

# Presentation request query types 

Talao build

## Motivation

When interacting with a Verifier it is likely that it wants to get a presentation made up of specific VCs. It is therefore necessary to be able to specify to the wallet the conditions to be applied to the choice of VCs. The following specifications are taken from a minimalist interpretation of the [W3C draft](https://w3c-ccg.github.io/vp-request-spec/#query-by-example) 

## Verifier implementation

There are 2 possibilities to foresee for the value of query.type of the JSON of the GET response of the Verify (“DIDAuth” or “QueryByExample”):

```javascript
{
           "type": "VerifiablePresentationRequest",
           "query": [{
               "type": “DIDAuth”
               }],
           "challenge": "a random uri",
           "domain" : "talao.co"
}
```

or: 


```javascript
{
           "type": "VerifiablePresentationRequest",
           "query": [{
               "type": "QueryByExample",
               "credentialQuery": [
                   {
                    ……
                   }]
               }],
           "challenge": "a random uri",
           "domain" : "talao.co"
 }
```

## DIDAuth

If Query.type = “DIDAuth” , then it is a basic authentication request that does not include a verifiable credential : there is no selection of credential to propose to the user, call the function didkit.DIDAuth(did, “{“ challenge ”:“ .... ”,“ domain ”:“ ..... ”}”, key) which will create an empty presentation used only for authentication. The presentation passed with the POST request will look like this:

```javascript
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1"
  ],
  "type": "VerifiablePresentation",
  "proof": {
    "type": "EcdsaSecp256k1Signature2019",
    "created": "2021-08-28T16: 13: 23.740Z",
    “challenge”: “d602e96d-08cb-11ec-a6fa-8d5c53eaebfb",
    “domain”: “talao.co”
    "jws ":" eyJhbGciOiJFUzI1NksiLCJjcml0IjpbImI2NCJdLCJiNjQiOmZhbHNlfQ..PgpEElB1tvcY9tdzK6EDKLvysj3vcH-zg5EIiGpk_q4m0NrAmjA81B7QdVvKllSzzfKw-1oTJuu4b4ihCvMXRwA
  "},
  "holder": "did:ethr:0xee09654eedaa79429f8d216fa51a129db0f72250"
}
```

If Query.type ="QueryByExample "then it will take the user selects credentials in a list constituted according to the criteria specified in "credentialQuery.example". Then it will be necessary to call the didkit.issuePresentation (...) function as what is currently done (there is no change in the function call). Refer to https://w3c-ccg.github.io/vp-request-spec/#query-by-example for more information.
    
## QueryByExample
    
### Overview

If "credentialQuery": is an empty list, one keeps the current behavior of Credible. The user is asked to select credentials to send. Never mind the VCs.

If "credentialQuery.example" contains {"reason": [......]}
then the Verifier wishes to display an information message to the user. This message will be displayed on the wallet at the time of selection.

If "credentialQuery.example" contains {"type": "some_type"}
then the Verifier wishes to receive VCs conforming to the specified type and the wallet presents a list of VCs consisting only of the specified type.

If "credentialQueryexample" contains { "trustedIssuer" : ["un_issuer", “un_autre_issuer”, ...]}
then the Verifier wishes to receive VCs sent by the specified Issuers and the wallet presents a list consisting only of the specified issuers.

Nota Bene : 
- There is one credentialQuery.example for each type of VC requested
- By default the credential is required ("required" : "True"), it does not support the other option.
- The reason attribute should be analysed as an array of different languages ("fr", "en", ...) 

#### QBE Examples

##### Example 1
Verifier requests VCs issued by did:tz:tz2NQkPq3FFA3zGAyG8kLcWatGbeXpHMu7yk:

```javascript
{
    "type": "VerifiablePresentationRequest",
    "query": [
        {
            "type": "QueryByExample",
            "credentialQuery": [
                {
                    "example" : {
                        "trustedIssuer": [
                            {
                                "issuer" : "did:tz:tz2NQkPq3FFA3zGAyG8kLcWatGbeXpHMu7yk"
                            }
                        ]
                    }
                }
            ]
        }
    ],
    "challenge": "9d0927c1-08cb-11ec-a6fa-8d5c53eaebfb",
    "domain": "talao.co"
}
```


##### Example 2
Verifier requests a ResidentCard:

```javascript
{
    "type": "VerifiablePresentationRequest",
    "query": [
        {
            "type": "QueryByExample",
            "credentialQuery": [
                {
                    "example" : {
                        "type" : "ResidentCard"
                    }
                }
            ]
        }
    ],
    "challenge": "9d0927c1-08cb-11ec-a6fa-8d5c53eaebfb",
    "domain": "talao.co"
}
```

##### Example 3
Verifier requests a ResidentCard and a DriverLicense and attaches messages for user :

```javascript
{
    "type": "VerifiablePresentationRequest",
    "query": [
        {
            "type": "QueryByExample",
            "credentialQuery": [
                {
                    "reason": [
                        {
                            "@language": "en",
                            "@value": "Join a resident card"
                        },
                        {
                            "@language": "fr",
                            "@value": "Joindre une carte de résidence"
                        }
                    ],
                    "example" : {
                        "type" : "ResidentCard"
                    }
                },
                {
                    "reason": [
                        {
                            "@language": "en",
                            "@value": "Join a driver license"
                        },
                        {
                            "@language": "fr",
                            "@value": "Joindre un permis de conduire"
                        }
                    ],
                    "example" : {
                        "type" : "DriverLicense"
                    }
                }
            ]
        }
    ],
    "challenge": "9d0927c1-08cb-11ec-a6fa-8d5c53eaebfb",
    "domain": "talao.co"
}

```

##### Example 4
Verifier attaches messages for user but no credential criters :

```javascript
{
    "type": "VerifiablePresentationRequest",
    "query": [
        {
            "type": "QueryByExample",
            "credentialQuery": [
                {
                    "reason": [
                        {
                            "@language": "en",
                            "@value": "Join a resident card and your driver license"
                        },
                        {
                            "@language": "fr",
                            "@value": "Joindre une carte de résidence et votre permis de conduire"
                        }
                    ]
                }
            ]
        }
    ],
    "challenge": "9d0927c1-08cb-11ec-a6fa-8d5c53eaebfb",
    "domain": "talao.co"
}

```

See https://talao.co/wallet/test/presentationRequest for simulation and testing.

# Universal link

The Talao wallet makes it possible to manage the interaction with an issuer or a verifier web application through a QR Code or a deeplink.

Depending on the origin of the request, we determine which device is accessing the web application.

## Access from a desktop viewer

Display a QR Code in the form of https: // my_endpoint? Issuer = my_did

## Access from smartphone viewer

Display a button link in the form of  https://app.talao.co/app/download?uri=https://my_endpoint?issuer=my_did

If the Talao wallet is not available on the smartphone, the user is referred to the page https://app.talao.co/app/download which offers him access to the Apple Store or the Google store. 



# Servers return codes
   
 ### 200 OK
 Color : Green  
 Message : No message or Credential presented successfully
 
 ### 201 Created
 Color : Green  
 Message : Ressource created
 
 ### 400 Bad Request
 Color : Red  
 Message : Bad request
 
 ### 401  unauthenticated
 Color : Red  
 Message :  The user must authenticate itself to get the requested response. 
 
 ### 403 Forbidden
 Color : Red  
 Message : Unauthorized request.  
 
 ### 408 Request Timeout
 Color : Red  
 Message : Request timeout
 
 ### 429 : Too many requests
 Color : Red  
 Message : The user has sent too many requests in a given amount of time.
 
 ### 500 Internal Server Error
 Color : Red  
 Message : This is a server internal error. Contact the server administrator.
 
 ### 501 Not Implemented
 Color : Red  
 Message : Not implemented 
 
 ### 504 Gateway Timeout
 Color : Red  
 Message : The gateway encountered a timeout

 ### other code
 Color : Red  
 Message : Received invalid status code
