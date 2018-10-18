---
title: Key Provisioning for Group Communication using ACE
abbrev: Key Prov Group Comm
docname: draft-palombini-ace-key-groupcomm-latest

ipr: trust200902
wg: ACE Working Group
cat: std

coding: utf-8
pi:
  toc: yes
  sortrefs: yes
  symrefs: yes
  tocdepth: 2

author:
      -
        ins: F. Palombini
        name: Francesca Palombini
        org: Ericsson AB
        street: Torshamnsgatan 23
        city: Kista
        code: SE-16440 Stockholm
        country: Sweden
        email: francesca.palombini@ericsson.com
      -
        ins: M. Tiloca
        name: Marco Tiloca
        org: RISE SICS
        street: Isafjordsgatan 22
        city: Kista
        code: SE-16440 Stockholm
        country: Sweden
        email: marco.tiloca@ri.se

normative:

  RFC2119:
  RFC8152:
  I-D.ietf-ace-oauth-authz:
  I-D.ietf-ace-oscore-profile:
  
informative:
  
  RFC7390:
  I-D.ietf-core-coap-pubsub:
  RFC2093:
  RFC2094:
  RFC2627:

  
--- abstract

This document defines a message format for distributing keying material in group communication scenarios (such as based on multicast or publisher-subscriber model) using the ACE framework.

--- middle

# Introduction {#intro}

This document expands the ACE framework {{I-D.ietf-ace-oauth-authz}} to define the format of messages used to distribute the keying material in a group communication scenario. Profiles that use group communication can build on this document to specify exactly which of the message parameters defined in this documents are used, and what are their values. Known applications that can benefit from this document would be, for example, profiles addressing group communication based on multicast {{RFC7390}} or publishing/subscribing {{I-D.ietf-core-coap-pubsub}} in ACE.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in {{RFC2119}}. These words may also appear in this document in lowercase, absent their normative meanings.

Readers are expected to be familiar with the terms and concepts described in  {{I-D.ietf-ace-oauth-authz}} and {{RFC8152}}.

# Overview

~~~~~~~~~~~
+-----------+                  +-----------+
|     AS    |                  |    KDC    |
|           |        .-------->|           |
+-----------+       /          +-----------+
      ^            / 
      |           /                  
      v          /                           +-----------+
+-----------+   /      +------------+        |+-----------+
|   Client  |<-'       | Dispatcher |        ||+-----------+
|           |<-------->|    (RS)    |<------->||   Group   |
+-----------+          +------------+         +|  members  |
                                               +-----------+

~~~~~~~~~~~
{: #fig-roles title="Key Distribution Participants" artwork-align="center"}

Participants:

* Client: Node that wants to join the group communication. It can either want write rights or read rights.
* AS: Same as AS in the ACE Framework; it contains policies, and knows if a node is allowed to join the group with write or read rights. 
* Key Distribution Center: Maintains the keying material to protect group communications, and provides it to clients authorized to join the group. During the first part of the exchange, it corresponds to the RS in the ACE Framework.
* Dispatcher: this is the entity the Client wants to securely communicate with and is responsible for distribution of group messages. It can be an existing node, such as the Broker in a pub-sub setting (in which case the Dispatcher is also a RS), or it can be implicit, as in the multicast communication setting, where the message distribution is done by transmitting to a multicast IP address, and entrusting message delivery to the transport channel.

<!-- Marco 22-02: A KDC can be responsible for more groups, while every group is associated to only one KDC.
FP: Proposal: let's add this sentence later. There is some considerations to be done about using a "cluster of KDC", but I don't want to overcomplicate v-00. Security considerations?
-->

This document specifies the message flows and formats for adding a node to a group, as well as for the distribution of keying material to joining nodes. Also, it briefly mentions the node's removal from a group and the consequent rekeying process.

The high level overview of the message flow for a node joining a group communication setting is shown in {{fig-flow}}.

~~~~~~~~~~~
C                 AS               KDC           Dispatcher
|                 |                 |                 | \
|  authorization  |                 |                 | |
|-----request---->|                 |                 | | defined in 
|                 |                 |                 | | the ACE
|  authorization  |                 |                 | | framework
|<----response----|                 |                 | |
|                 |                 |                 | |
|--------token post---------------->|                 | /
|                 |                 |                 |
|----key distribution request------>|                 |
|                 |                 |                 |
|<---key distribution response------|                 |
|                 |                 |                 |
|<=============protected communication===============>| 
|                 |                 |                 |
~~~~~~~~~~~
{: #fig-flow title="Key Distribution Message Flow" artwork-align="center"}

# Addition to the Group

This section describes in detail the message formats exchanged by the participants when a node requests access to the group. The first part of the exchange is based on ACE {{I-D.ietf-ace-oauth-authz}}, where the KDC takes the role of RS.

## Authorization Request

The Authorization Request sent from the Client to the AS (as defined in {{I-D.ietf-ace-oauth-authz}}, Section 5.6.1, MUST contain the following parameters:

* grant_type, with value "client_credentials".

Additionally, the Authorization Request MAY contain the following parameters, which, if included, MUST have the corresponding values:


<!-- 
  Based on a discussion with Ludwig, the "scope" claim represented as a CBOR Array should be further encoded as CBOR byte array, to be compliant with the ACE framework.
-->

* scope, with value the identifier of the specific group or topic the Client wishes to access, as well as the role the Client wishes to take, if necessary. This value is a CBOR array encoded as a byte string, which contains:

  - as first element, the identifier of the specific group or topic
  - optionally, as second element, the role (or CBOR array of roles) the Client wishes to take in the group

  How exactly the group or topic identifier and the roles are encoded is application specific.

* aud, with value an identifier of the KDC.

* cnf, containing the public key (or certificate) of the Client, if it wishes to communicate that to the AS.

<!--
  Jim 14-06: Remove "get_pub_keys" from the Authorization Request and Access Token. There is no particular reason why it should be here, i.e. that the AS cases at all about it.  
-->

* Other additional parameters as defined in {{I-D.ietf-ace-oauth-authz}}, if necessary. 

<!--
Marco 27-02: “scope” should include a list of identifiers. One can ask authorization for joining multiple groups in a single Authorization Request, so getting a single Access Token.

FP: Is this something we really want to cover? I think this could open up to a number of comments and questions (how do you renew keying material for just one of these res, for example). Let's think a bit more about this.
-->

## Authorization Response

The Authorization Response sent from the AS to the Client (as defined in {{I-D.ietf-ace-oauth-authz}}, Section 5.6.2, MUST contain the following parameters:

* access_token, containing all the parameters defined below (including the same 'scope' as in this message, if present, or the 'scope' of the Authorization Request otherwise), and additionally other optional parameters the profile requires.

* cnf if symmetric keys are used, not present if asymmetric keys are used, contains the symmetric pop key that the Client is supposed to use with the KDC.

* rs_cnf if asymmetric keys are used, contains information about the public key of the KDC. Not present if symmetric keys are used.

* exp, contains the lifetime in seconds of the access token. This parameter MAY be omitted if the application defines how the expiration time is communicated to the Client via other means, or if it establishes a default value.

Additionally, the Authorization Response MAY contain the following parameters, which, if included, MUST have the corresponding values:

* scope, which mirrors the 'scope' parameter in the Authorization Request {{authorization-request}}. Its value is a CBOR array encoded as a byte string, containing:

  - as first element, the identifier of the specific group or topic the Client is authorized to access.

  - optionally, as second element, the role (or CBOR array of roles) the Client is authorized to take in the group.

  How exactly the group or topic identifier and the roles are encoded is application specific.

* Other additional parameters as defined in {{I-D.ietf-ace-oauth-authz}}, if necessary.

<!--
  Jim 14-06: State explicitly what the AS does when receiving an authorization request for a group from a joining node that already has a valid not expired token to join that group (and is probably an actual member). The AS can simply issue a new token.

  TODO: Note that if the joining node valid not expired token (for example if it was previously member of the group), the AS still issue a new token.

-->

When receiving an Authorization Request from a Client that was previously authorized, and which still owns a valid non expired Access Token, the AS can simply reply with an Authorization Response including a new Access Token.

## Token Post

The Client sends a CoAP POST request including the Access Token to the KDC, as specified in section 5.8.1 of {{I-D.ietf-ace-oauth-authz}}. If the specific profile defines it, the Client MAY use a different endpoint at the KDC to post the Access Token to. After successful verification, the Client is authorized to receive the group keying material from the KDC and join the group.

Note that this step could be merged with the following message from the Client to the KDC, namely Key Distribution Request.

# Key Distribution

This section defines how the keying material used for group communication is distributed from the KDC to the Client, when joining the group as a new member.

The same types of messages can also be used for the following cases, when the Client is already a group member:

* The Client wishes to (re-)get the current keying material, for cases such as expiration, loss or suspected mismatch, due to e.g. reboot or missed rekeying. This is further discussed in {{sec-expiration}}.

* The Client wishes to (re-)get the public keys of other group members, e.g. if it is aware of new nodes joining the group after itself. This is further discussed in {{sec-key-retrieval}}.

<!--
  Jim 14-06: Discuss that a Key Distribution Request/Response can be performed exactly in the same way also by an already member of the group. Mention the cases when this happens, e.g. believed lost of synchronization with the current group security context, crash and reboot and so on, so forced re-synchronization with the correct current security context.

  TODO: Add a general description on when the following msgs are used:
    - join new nodes
    - member for rekeying (triggered by KDC)
    - member after they forgot (crash)
-->

## Key Distribution Request {#ssec-key-distribution-request}

The Client sends a Key Distribution request to the KDC.
This corresponds to a CoAP POST request to the endpoint in the KDC associated to the group (which is associated in the KDC to the 'scope' value of the Authorization Request/Response). The payload of this request is a CBOR Map which MAY contain the following fields, which, if included, MUST have the corresponding values:

* scope, with value the specific resource or topic identifier and role(s) that the Client is authorized to access, encoded as in {{authorization-request}}.

<!--
  Jim 14-06: We need an API for distributing all public keys or a specific public key to an endpoint already member of the group. In the first case, the query information can be the identifier of the group/topic. In the second case, the query information can be the endpoint ID associated to the key to be retrieved, as considered as key identifier. This particular details such as "Group ID" and "Sender ID" are specified in the main group OSCORE document as a particular instance of this generic model.

  TODO: define format get_pub_keys: []
-->

* get_pub_keys, if the Client wishes to receive the public keys of the other nodes in the group from the KDC. The value is an empty CBOR Array. This parameter may be present if the KDC stores the public keys of the nodes in the group and distributes them to the Client; it is useless to have here if the set of public keys of the members of the group is known in another way, e.g. it was supplied by the AS.

* client_cred, with value the public key or certificate of the Client. If the KDC is managing (collecting from/distributing to the Client) the public keys of the group members, this field contains the public key of the Client.

* pub_keys_repos, can be present if a certificate is present in the client_cred field, with value a list of public key repositories storing the certificate of the Client.

## Key Distribution Response {#ssec-key-distribution-response}

The KDC verifies the Access Token and, if verification succeeds, sends a Key Distribution success Response to the Client. This corresponds to a 2.01 Created message. The payload of this response is a CBOR Map which MUST contain the following fields:

* key, used to send the keying material to the Client, as a COSE_Key ({{RFC8152}}) containing the following parameters:
  - kty, as defined in {{RFC8152}}.
  - k, as defined in {{RFC8152}}.
  - exp (optionally), as defined below. This parameter is RECOMMENDED to be included in the COSE_Key. If omitted, the authorization server SHOULD provide the expiration time via other means or document the default value.
  - alg (optionally), as defined in {{RFC8152}}.
  - kid (optionally), as defined in {{RFC8152}}.
  - base iv (optionally), as defined in {{RFC8152}}.
  - clientID (optionally), as defined in {{I-D.ietf-ace-oscore-profile}}.
  - serverID (optionally), as defined in {{I-D.ietf-ace-oscore-profile}}.
  - kdf (optionally), as defined in {{I-D.ietf-ace-oscore-profile}}.
  - slt (optionally), as defined in {{I-D.ietf-ace-oscore-profile}}.
  - cs_alg (optionally), containing the algorithm value to countersign the message, taken from Table 5 and 6 of {{RFC8152}}.

<!--
TODO: Add exp in COSE_Key = same as exp in token but for the key
define it as a COSE Key Common Parameter (see section 7.1 of COSE)
-->

The parameter 'exp' identifies the expiration time in seconds after which the COSE\_Key is not valid anymore for secure communication in the group. A summary of 'exp' can be found in {{table-additional-param}}.

~~~~~~~~~~~
+------+-------+----------------+------------+-----------------+
| Name | Label | CBOR Type      | Value      | Description     |
|      |       |                | Registry   |                 |
+------+-------+----------------+------------+-----------------+
| exp  | TBD   | Integer or     | COSE Key   | Expiration time |
|      |       | floating-point | Common     | in seconds      |
|      |       | number         | Parameters |                 |
+------+-------+----------------+------------+-----------------+
~~~~~~~~~~~
{: #table-additional-param title="COSE Key Common Header Parameter 'exp'" artwork-align="center"}

Additionally, the Key Distribution Response MAY contain the following parameters, which, if included, MUST have the corresponding values:

* pub\_keys, may only be present if get\_pub\_keys was present in the Key Distribution Request; this parameter is a COSE\_KeySet (see {{RFC8152}}), which contains the public keys of all the members of the group.

* group_policies, with value a list of parameters indicating how the group handles specific management aspects. This includes, for instance, approaches to achieve synchronization of sequence numbers among group members. The exact format of this parameter is specific to the profile.

* mgt_key_material, with value the administrative keying material to participate in the revocation and renewal of group keying (rekeying) performed by the KDC. The exact format and content depend on the specific rekeying algorithm used in the group, which may be specified in the profile.

Specific profiles need to specify how exactly the keying material is used to protect the group communication.

TBD: define for verification failure

# Remove a Node from the Group

This section describes at a high level how a node can be removed from the group.

## Not authorized anymore

If the node is not authorized anymore, the AS can directly communicate that to the KDC. Alternatively, the Access Token might have expired. If Token introspection is provided by the AS, the KDC can use it as per Section 5.7 of {{I-D.ietf-ace-oauth-authz}}, in order to verify that the Access Token is still valid.

Either case, once aware that a node is not authorized anymore, the KDC has to generate and distribute the new keying material to all authorized members of the group, as well as to remove the unauthorized node from the list of members (if the KDC keeps track of that). The KDC relies on the specific rekeying algorithm used in the group, such as e.g. {{RFC2093}}, {{RFC2094}} or {{RFC2627}}, and the related management key material.

## Request to Leave the Group

A node can actively request to leave the group. In this case, the Client can send a request to the KDC to exit the group. The KDC can then generate and distribute the new keying material to all authorized members of the group, as well as remove the leaving node from the list of members (if the KDC keeps track of that).

Note that, as long as the node is authorized to join the group, i.e. it has a valid Access Token, it can re-request to join the group directly to the KDC without needing to retrieve a new Access Token. This means that the KDC needs to keep track of nodes with valid Access Tokens, before deleting all information about the leaving node.

# Retrieval of Updated Keying Material {#sec-expiration}

A node stops using the group keying material upon its expiration, according to the 'exp' parameter specified in the retained COSE Key. Then, if it wants to continue participating in the group communication, the node has to request new updated keying material to the KDC.

The Client may perform the same request to the KDC also upon receiving messages from other group members without being able to correctly decrypt them. This may be due to a previous update of the group keying material (rekeying) triggered by the KDC, that the Client was not able to participate to.

Note that policies can be set up so that the Client sends a request to the KDC only after a given number of unsuccessfully decrypted incoming messages.

## Key Re-Distribution Request

To request a re-distribution of keying material, the Client sends a shortened Key Distribution request to the KDC ({{ssec-key-distribution-request}}), formatted as follows. The payload MAY contain only the following field:

* scope, which contains only the identifier of the specific group or topic, encoded as in {{authorization-request}}. That is, the role field is not present.

In some cases, it is not necessary to include the scope parameter, for instance if the KDC maintains a list of active group members for each managed group, and the Client is member of only one group. The Client MUST include the scope parameter if it is a member of multiple groups under the same KDC.

## Key Re-Distribution Response

The KDC replies to the Client with a Key Distribution Response containing the 'key' parameter, and optionally 'group_policies' and 'mgt_key_material', as specified in {{ssec-key-distribution-response}}. Note that this response might simply re-provide the same keying material currently owned by the Client, if it has not been renewed.

# Retrieval of Public Keys for Group Members {#sec-key-retrieval}

In case the KDC maintains the public keys of group members, a node in the group can contact the KDC to request public keys of either all group members or a specified subset, using the messages defined below.

Note that these messages can be combined with the Key Re-Distribution messages in {{sec-expiration}}, to request at the same time the keying material and the public keys. In this case, either a new endpoint at the KDC may be used, or additional information needs to be sent in the request payload, to distinguish these combined messages from the Public Key messages described below, since they would be identical otherwise.

## Public Key Request

To request public keys, the Client sends a shortened Key Distribution request to the KDC ({{ssec-key-distribution-request}}), formatted as follows. The payload of this request MUST contain the following field:

* get_pub_keys, which has for value a CBOR array including either:
  - no elements, i.e. an empty array, in order to request the public key of all current group members; or
  - N elements, each of which is the identifier of a group member, in order to request the public key of the specified nodes.

Additionally, this request MAY contain the following parameter, which, if included, MUST have the corresponding value:

* scope, which contains only the identifier of the specific group or topic, encoded as in {{authorization-request}}. That is, the role field is not present.

  In some cases, it is not necessary to include the scope parameter, for instance if the KDC maintains a list of active group members for each managed group, and if the specified identifiers allow to retrieve public keys with no ambiguity. The Client MUST include the scope parameter if it is a member of multiple groups under the same KDC.


If the KDC can not unambiguously identify the nodes specified in the 'get_pub_keys' parameter, it MUST reply with an error message. In this case, the Client can issue a new Public Key Request specifying the group in the 'scope' parameter.

TODO: define error

## Public Key Response

The KDC replies to the Client with a Key Distribution Response containing only the 'pub_keys' parameter, as specified in {{ssec-key-distribution-response}}. The payload of this response contains the following field:

* pub_keys, which contains either:

  - the public keys of all the members of the group, if the 'get_pub_keys' parameter of the Public Key request was an empty array; or

  - the public keys of the group members with the identifiers specified in the 'get_pub_keys' parameter of the Public Key request.

The KDC ignores possible identifiers included in the 'get_pub_keys' parameter of the Public Key request if they are not associated to any current group member.

# Security Considerations

The KDC must renew the group keying material upon its expiration.

The KDC should renew the keying material upon group membership change, and should provide it to the current group members through the rekeying algorithm used in the group.

# IANA Considerations

The following registration is required for the COSE Key Common Parameter Registry specified in Section 16.5 of {{RFC8152}}:

*  Name: exp
*  Label: TBD
*  CBOR Type: Integer or floating-point number
*  Value Registry: COSE Key Common Parameters
*  Description: Identifies the expiration time in seconds of the COSE Key
*  Reference: \[\[this specification\]\]

--- back

# Acknowledgments
{: numbered="no"}

The following individuals were helpful in shaping this document: Ben Kaduk, John Mattsson, Jim Schaad, Ludwig Seitz and Göran Selander.

The work on this document has been partly supported by the EIT-Digital High Impact Initiative ACTIVE.

--- fluff
