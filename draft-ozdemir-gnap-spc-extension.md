---
title: "GNAP Secure Payment Confirmation Extension"
category: info

docname: draft-ozdemir-gnap-spc-extension-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Grant Negotiation and Authorization Protocol"
keyword:
 - gnap
 - secure payment confirmation
 - extension
venue:
  group: "Grant Negotiation and Authorization Protocol"
  github: "fynbos-dev/gnap-spc-extension"
  latest: "https://fynbos-dev.github.io/gnap-spc-extension/draft-ozdemir-gnap-spc-extension.html"

author:
 -
    fullname: Omer Talha Ozdemir
    organization: Fynbos
    email: omer@fynbos.dev
 -
    fullname: Adrian Hope-Bailie
    organization: Fynbos
    email: adrian@fynbos.dev

normative:
  GNAP: I-D.ietf-gnap-core-protocol
  SPC: W3C.secure-payment-confirmation
  WebAuthn: W3C.webauthn-3

informative:
  PaymentRequest: W3C.payment-request-1.1

entity:
  SELF: "RFC nnnn"

--- abstract

GNAP Secure Payment Confirmation (SPC) Extension is a Grant Negotiation and Authorization Protocol ({{GNAP}}) extension that defines a method for authentication of the end user during a payment transaction. This extension helps leverage hardware and software authenticators such as biometric scanners while authenticating the end user.

--- middle

# Introduction

GNAP Secure Payment Confirmation Extension is an extension developed on top of the Grant Negotiation and Agreement Protocol {{GNAP}}. It defines a method for authentication of the end user during a payment transaction using Secure Payment Confirmation ({{SPC}}). This extension helps leverage authenticators such as fingerprint scanners, facial recognition systems, etc. while authenticating during a GNAP interaction.

A method for detecting this capability in the client software is provided in {{detect-spc}}.

## Conventions and Definitions

{::boilerplate bcp14-tagged}


# Secure Payment Confirmation Interaction

When using SPC in {{GNAP}}, the end user is prompted to authenticate during the interaction phase of the protocol, when the grant is in the _pending_ state.

The overall flow of the protocol is shown here:

~~~ aasvg
+--------+                                  +--------+
| Client |                                  |   AS   |
|Instance|                                  |        |
|        |                                  |        |
|        +--(1)--- Request Access --------->|        |
|        |                                  |        |
|        |<-(2)-- Interaction Needed -------+        |
|        |                                  |        |
|        |           .----.                 |        |
|        |          | End  |                |        |
|        |          | User |                |        |
|        |<==(3)===>|------+                |        |
|        |   SPC    |  RO  |                |        |
|        |          |      |                |        |
|        |           `----`                 |        |
|        |                                  |        |
|        +--(4)--- Continue Request ------->|        |
|        |                                  |        |
|        |                              ,---+        |
|        |                         (5) |    |        |
|        |                         SPC  `-->|        |
|        |                                  |        |
|        |<-(6)----- Grant Access ----------+        |
|        |                                  |        |
+--------+                                  +--------+
~~~

1. The client instance makes a grant request to the AS, indicating that it can support SPC as an interaction start method. The client instance includes an identifier for the intended end user in the request. The client instance does not include an interaction finish method, since the interaction will happen outside of the AS and the client software will not need to be signaled by the AS to continue the grant request. ({{request-credentials}})

2. The AS creates a new grant request in the _pending_ state and responds to the grant request with a challenge to be signed by a set of candidate credentials. ({{serve-credentials}})

3. The client instance engages the SPC protocol to sign the challenge. ({{authenticate-user}})

4. The client instance continues the grant request, including the results of the SPC challenge response from (3) in the grant continuation request. ({{complete-interaction}})

5. The AS validates the signature, completing the SPC process and placing the grant in the _approved_ state. ({{verifying-authentication}})

6. The AS returns an access token in a standard GNAP grant response.


The end user provides confirmation using their credential, which the client instance presents back to the AS in a continuation response. The AS then verifies this confirmation in order to process the grant request and grant access to the client instance.


## Requesting Credentials {#request-credentials}

A client instance that is able to use SPC for user interaction can request a grant from an authorization server with SPC as an interaction start method.

The SPC interaction start method is defined as a string type with value `spc`. If the `spc` interaction start method is accepted, the corresponding response is returned as discussed in {{serve-credentials}}.

When requesting the SPC interaction method the client MUST identify the end user using the `user` property in the grant request object to allow the AS to find the registered credentials for the user. If the `user` property is not included in the request, the AS MUST NOT enable this interaction mode for this request.

A non-normative example of a grant request that uses SPC as its interaction start method is below.

~~~ json
"access_token": {
  "access": ["make-payment"]
},
"client": "xyz-client-1234a",
"interact": {
  "start": [
    "spc"
  ]
},
"user": {
  "sub_ids": [{
    "subject_type": "email",
    "email": "user@example.com"
  }]
}
~~~

## Providing a Credential Challenge {#serve-credentials}

In response to a client instance’s grant request, if the AS determines that it has a registered SPC credential of the end user, the AS responds with an `spc` field in the `interact` object.

The AS determines the end user using `user` property from the grant request. `user` property should have the required information such as email addresses, usernames, etc. for determining the end user, see {{Section 2.4 of GNAP}}. If the `user` property is not included in the request, it is not possible for the AS to determine the credentials for the end user.

`spc` (object):
: An object containing parameters required for performing secure payment confirmation on the end user's device. REQUIRED if the AS is allowing SPC interaction for this request.

This object contains the following properties:

`credential_ids` (array of strings):
: A list of identifiers of credentials that could potentially be used to respond to this interaction mode. Each credential identifier MUST be base64url encoded with no padding. REQUIRED.

`challenge` (string):
: A random challenge that the relying party generates on the server side to prevent replay attacks. The challenge MUST be base64url encoded with no padding. REQUIRED.

A non-normative example of a grant request continue response that uses SPC as its interaction method is below.

~~~ json
{
  "interact": {
    "spc": {
      "credential_ids": ["MTIzMjMxMzIyMz..."],
      "challenge": "dGhpcyBpcyBh..."
    }
  },
  "continue": {
    "access_token": {
      "value": "80UPRY5NM33OMUKMKSKU"
    },
    "uri": "http://wallet.com/adrian/continue/5e69f364-b14d-4fdf-8b6b-3b6ffb52c339"
  }
}
~~~

## Authenticating User {#authenticate-user}

When the client instance receives an `spc` interaction response from the AS, the client instance SHOULD initiate the authentication ceremony following Section 4 of {{SPC}}.

When performing this ceremony, the client instance decodes the `challenge` and each credential from `credential_ids` using base64url and convert them to a buffer for input into the browser API. When the authentication ceremony is complete, the client instance will have access to the response data from the ceremony to be returned to the AS.

Each credential id in `credential_ids` property the AS provided is registered by the end user in the past. When the client initates the authentication ceremony, the browser API is going to check if the device has at least one of the credential ids and continue to the authentication ceremony only if the device has one of the credentials. In this phase, the credential that end user choose as authentication method is going to be used for signing the cryptogram.

## Completing Interaction {#complete-interaction}

Once the authentication ceremony is complete, the client instance continues the grant request by calling the grant continuation URI. The client instance includes a body in the grant continuation request including a field named `public_key_cred`:

`public_key_cred` (object):
: The results of the SPC authentication ceremony in response to the AS, to be used by the AS to authorize the request.

The `public_key_cred` object contains the following fields as defined by the Web Authentication Assertion object {{WebAuthn}}:

`clientDataJSON` (string):
: `clientDataJSON` property from Web Authentication Assertion object. This **MUST** be encoded using base64url. **REQUIRED**.

`authenticatorData` (string):
: `authenticatorData` property from Web Authentication Assertion object. This **MUST** be encoded using base64url. **REQUIRED**.

`signature` (string):
: `signature` property from Web Authentication Assertion object. This **MUST** be encoded using base64url. **REQUIRED**.

`userHandle` (string):
: `userHandle` property from Web Authentication Assertion object. This **MUST** be encoded using base64url. **REQUIRED**.

A non-normative example of an interaction completion response body is below.

~~~ json
{
  "public_key_cred": {
    "clientDataJSON": "ZXhhbXBsZSBjbGllbnRkYXR...",
    "authenticatorData": "YXV0aGVudGljYXRvckRhdGEg...",
    "signature": "c2lnbmF0dXJlIGV4YW...",
    "userHandle": "dXNlckhhbmRsZSBleG..."
  }
}
~~~

Since the signature is in response to a challenge provided by the AS, the client instance MUST NOT send this parameter to a new grant request. The grant request MUST be in the _pending_ state when this parameter is sent to the AS.

# Verifying Authentication Assertion {#verifying-authentication}

When the AS receives the `public_key_cred` value in a grant continuation request, the AS MUST perform the steps specified in Section 8.1 of {{SPC}}. Each property of a public key credential returned successful invocation of the SPC handler `clientDataJSON`, `authenticatorData`, `signature` and `userHandle` MUST be present as expected for starting verification. The AS MUST decode each property of the public key credential in the response using base64url before performing the verification.

The grant request MUST be in the _pending_ state when this parameter is received in order for it to be processed. If the grant request is in any other state, the AS MUST return an error.

The AS MUST ensure that the transaction details encoded in the public key credential match the details of the transaction that the client instance is requesting a grant to perform.

If the AS determines that the authorization is sufficient, the AS grants access tokens and releases subject information to the client instance.

# Security Considerations

TODO Security


# IANA Considerations

IANA is requested to register the following values into the named registries.

## Interaction Start Modes

IANA is requested to register the following modes into the Interaction Start Modes registry defined by {{GNAP}}.

|Mode|Type|Specification document(s)|
|spc|string|{{request-credentials}} of {{&SELF}}|

## Interaction Mode Responses

IANA is requested to register the following methods into the Interaction Mode Responses registry defined by {{GNAP}}.

|Name|Specification document(s)|
|spc|{{serve-credentials}} of {{&SELF}}|


## Grant Request Parameters

IANA is requested to register the following parameters into the Grant Request Parameters registry defined by {{GNAP}}.

|Name|Type|Specification document(s)|
|public_key_cred|object|{{complete-interaction}} of {{&SELF}}|


# Acknowledgments
{:numbered="false"}

TODO acknowledge.

--- back

# Checking Feature Support {#detect-spc}

This extension only works if the end user's user agent supports the Payment Request API {{PaymentRequest}} and SPC. To detect whether SPC is supported on the browser, the client instance can send a fake call to `canMakePayment()`.

The following code provides a feature detect function for the Payment Request API and SPC that could be executed on a merchant's website.

~~~ javascript
const isSecurePaymentConfirmationSupported = async () => {
  if (!'PaymentRequest' in window) {
    return [false, 'Payment Request API is not supported'];
  }

  try {
    // The data below is the minimum required to create the request and
    // check if a payment can be made.
    const supportedInstruments = [
      {
        supportedMethods: "secure-payment-confirmation",
        data: {
          // RP's hostname as its ID
          rpId: 'rp.example',
          // A dummy credential ID
          credentialIds: [new Uint8Array(1)],
          // A dummy challenge
          challenge: new Uint8Array(1),
          instrument: {
            // Non-empty display name string
            displayName: ' ',
            // Transparent-black pixel.
            icon: 'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+P+/HgAFhAJ/wlseKgAAAABJRU5ErkJggg==',
          },
          // A dummy merchant origin
          payeeOrigin: 'https://non-existent.example',
        }
      }
    ];

    const details = {
      // Dummy shopping details
      total: {label: 'Total', amount: {currency: 'USD', value: '0'}},
    };

    const request = new PaymentRequest(supportedInstruments, details);
    const canMakePayment = await request.canMakePayment();
    return [canMakePayment, canMakePayment ? '' : 'SPC is not available'];
  } catch (error) {
    console.error(error);
    return [false, error.message];
  }
};
~~~

