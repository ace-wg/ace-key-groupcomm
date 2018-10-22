---
title: Key Provisioning for Group Communication using ACE
abbrev: Key Provisioning for Group Communication
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
        org: RISE AB
        street: Isafjordsgatan 22
        city: Kista
        code: SE-16440 Stockholm
        country: Sweden
        email: marco.tiloca@ri.se

normative:

  RFC2119:
  RFC8152:
  I-D.ietf-ace-oauth-authz:
  I-D.ietf-ace-oauth-params:
  I-D.ietf-ace-oscore-profile:
  
informative:
  
  RFC7390:
  I-D.ietf-core-coap-pubsub:
  RFC2093:
  RFC2094:
  RFC2627:

  
--- abstract

This document defines message formats and procedures for requesting and distributing group keying material using the ACE framework, to protect communications between group members.

--- middle

# Introduction {#intro}

This document expands the ACE framework {{I-D.ietf-ace-oauth-authz}} to define the format of messages used to request, distribute and renew the keying material in a group communication scenario, e.g. based on multicast {{RFC7390}} or on publishing-subscribing {{I-D.ietf-core-coap-pubsub}}.

Profiles that use group communication can build on this document to specify the selection of the message parameters defined in this document to use and their values. Known applications that can benefit from this document would be, for example, profiles addressing group communication based on multicast {{RFC7390}} or publishing/subscribing {{I-D.ietf-core-coap-pubsub}} in ACE.

If the application requires backward and forward security, updated keying material is generated and distributed to the group members (rekeying), when membership changes. A key management scheme performs the actual distribution of the updated keying material to the group. In particular, the key management scheme rekeys the current group members when a new node joins the group, and the remaining group members when a node leaves the group. This document provides a message format for group rekeying that allows to fulfill these requirements. Rekeying mechanisms can be based on {{RFC2093}}, {{RFC2094}} and {{RFC2627}}.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in {{RFC2119}}. These words may also appear in this document in lowercase, absent their normative meanings.

Readers are expected to be familiar with the terms and concepts described in  {{I-D.ietf-ace-oauth-authz}} and {{RFC8152}}, such as Authorization Server (AS) and Resource Server (RS).


# Overview

~~~~~~~~~~~
+------------+                  +-----------+
|     AS     |                  |    KDC    |
|            |        .-------->|           |
+------------+       /          +-----------+
      ^             / 
      |            /                  
      v           /                           +-----------+
+------------+   /      +------------+        |+-----------+
|   Client   |<-'       | Dispatcher |        ||+-----------+
|            |<-------->|    (RS)    |<------->||   Group   |
+------------+          +------------+         +|  members  |
                                                +-----------+
~~~~~~~~~~~
{: #fig-roles title="Key Distribution Participants" artwork-align="center"}

<!-- Peter 30-07: Figure 1 is not referenced in the text. I suggest a slightly different figure where dispatcher and KDC are endpoints of the RS, and for multicast the communication is directly between Client and group members without passing through the RS.

Marco: I am not sure the change is consistent, since the KDC is an AS in pub-sub, i.e. not an RS or part of an RS. Also, in group OSCORE the KDC is the RS, not part of it.

Marco: We have extended the definition of Dispatcher, clarifying the two main cases involving either a Broker (ACE RS) or a bus (multicast delivery). Does this help?
-->

The following participants (see {{fig-roles}}) take part in the authorization and key distribution.

* Client (C): node that wants to join the group communication. It can request write and/or read rights.

* Authorization Server (AS): same as AS in the ACE Framework; it enforces access policies, and knows if a node is allowed to join the group with write and/or read rights. 

* Key Distribution Center (KDC): maintains the keying material to protect group communications, and provides it to Clients authorized to join the group. During the first part of the exchange ({{sec-auth}}), it takes the role of the RS in the ACE Framework. During the second part ({{key-distr}}), which is not based on the ACE Framework, it distributes the keying material. In addition, it provides the latest keying material to group members when requested. If required by the application, the KDC renews and re-distributes the keying material in the group when membership changes.

* Dispatcher: entity through which the Clients communicate with the group and which distributes messages to the group members. Examples of dispatchers are: the Broker node in a pub-sub setting; a relayer node for group communication that delivers group messages as multiple unicast messages to all group members; an implicit entity as in a multicast communication setting, where messages are transmitted to a multicast IP address and delivered on the transport channel.

<!-- Marco 22-02: A KDC can be responsible for more groups, while every group is associated to only one KDC.
FP: Proposal: let's add this sentence later. There is some considerations to be done about using a "cluster of KDC", but I don't want to overcomplicate v-00. Security considerations?
-->

This document specifies the message flows and formats for:

* Authorizing a new node to join the group ({{sec-auth}}), and providing it with the group keying material to communicate with the other group members ({{key-distr}}).

* Removing of a current member from the group ({{sec-node-removal}}).

* Retrieving keying material as a current group member ({{sec-expiration}} and {{sec-key-retrieval}}).

* Renewing and re-distributing the group keying material (rekeying) upon a membership change in the group ({{ssec-key-distribution-response}} and {{sec-node-removal}}).

{{fig-flow}} provides a high level overview of the message flow for a node joining a group communication setting.

~~~~~~~~~~~
C                              AS     KDC   Dispatcher          Group
|                              |       |        |               Member
|                              |       |        | \               |
|     Authorization Request    |       |        | | Defined       |
|----------------------------->|       |        | | in the ACE    |
|                              |       |        | | framework     |
|     Authorization Response   |       |        | |               |
|<-----------------------------|       |        | |               |
|                              |       |        | |               |
|--------- Token Post ---------------->|        | /               |
|                                      |        |                 |
|---- Key Distribution Request ------->|        |                 |
|                                      |        |                 |
|<--- Key Distribution Response ------ | --- Group Rekeying ----->|
|                                               |                 |
|<================== Protected communication ===|================>|
|                                               |                 |
~~~~~~~~~~~
{: #fig-flow title="Message Flow Upon New Node's Joining" artwork-align="center"}

The exchange of Authorization Request and Authorization Response between Client and AS MUST be secured, as specified by the ACE profile used between Client and KDC.

The exchange of Key Distribution Request and Key Distribution Response between Client and KDC MUST be secured, as a result of the ACE profile used between Client and KDC. 

All further communications between the Client and the KDC MUST be secured, for instance with the same security mechanism used for the Key Distribution exchange.

All further communications between a Client and the other group members MUST be secured using the keying material provided in {{key-distr}}.

# Authorization to Join a Group {#sec-auth}

This section describes in detail the format of messages exchanged by the participants when a node requests access to a group. The first part of the exchange is based on ACE {{I-D.ietf-ace-oauth-authz}}.

As defined in {{I-D.ietf-ace-oauth-authz}}, the Client requests from the AS an authorization to join the group through the KDC (see {{ssec-authorization-request}}). If the request is approved and authorization is granted, the AS provides the Client with a proof-of-possession access token and parameters to securely communicate with the KDC (see {{ssec-authorization-response}}). Communications between the Client and the AS MUST be secured, and depends on the profile of ACE used.

{{fig-group-member-registration}} gives an overview of the exchange described above.

~~~~~~~~~~~
Client                                            AS  KDC
   |                                               |   |
   |---- Authorization Request: POST /token ------>|   |
   |                                               |   |
   |<--- Authorization Response: 2.01 (Created) ---|   |
   |                                               |   |
   |----- POST Token: POST /authz-info --------------->|
   |                                                   |
~~~~~~~~~~~
{: #fig-group-member-registration title="Message Flow of Join Authorization" artwork-align="center"}

<!-- Peter 30-07: It would be nice if here the use of DTLS (or not) and the content format is specified: application/cbor or application/Cose+cbor

[MT] This should be out of scope, as actually per the specific ACE profile in use.
-->

## Authorization Request {#ssec-authorization-request}

The Authorization Request sent from the Client to the AS is as defined in Section 5.6.1 of {{I-D.ietf-ace-oauth-authz}} and MUST contain the following parameters:

* 'grant_type', with value "client_credentials".

Additionally, the Authorization Request MAY contain the following parameters, which, if included, MUST have the corresponding values:

* 'scope', with value the identifier of the specific group or topic the Client wishes to access, and optionally the role(s) the Client wishes to take. This value is a CBOR array encoded as a byte string, which contains:

  - As first element, the identifier of the specific group or topic.

  - Optionally, as second element, the role (or CBOR array of roles) the Client wishes to take in the group.

  The encoding of the group or topic identifier and of the role identifiers is application specific.

* 'req_aud', as defined in Section 3.1 of {{I-D.ietf-ace-oauth-params}}, with value an identifier of the KDC.

* 'req_cnf', as defined in Section 3.1 of {{I-D.ietf-ace-oauth-params}}, optionally containing the public key or the certificate of the Client, if it wishes to communicate that to the AS.

<!-- Peter 30-07: Question: is this a certificate identifier, or the public key extracted from the certificate, or a hash?????

Marco: It is just as per ACE. See Sections 3.2 and 3.4 of draft-ietf-ace-cwt-proof-of-possession-03
-->

* Other additional parameters as defined in {{I-D.ietf-ace-oauth-authz}}, if necessary. 

<!--
Marco 27-02: “scope” should include a list of identifiers. One can ask authorization for joining multiple groups in a single Authorization Request, so getting a single Access Token.

Jim 13-07: Section 3.1 - Can I get authorization for multiple items at a single time?

FP: Is this something we really want to cover? I think this could open up to a number of comments and questions (how do you renew keying material for just one of these res, for example). Let's think a bit more about this.
-->

<!-- Jim 13-07: Section 3.1 - Does it make sense to allow for multiple audiences to be on a single KDC?  

Marco: In principle yes, if you consider a logical audience as the GM/Broker at a single physical KDC.

Should we discuss this in the draft?
-->

## Authorization Response {#ssec-authorization-response}

The Authorization Response sent from the AS to the Client is as defined in Section 5.6.2 of {{I-D.ietf-ace-oauth-authz}} and MUST contain the following parameters:

* 'access_token', containing the proof-of-possession access token. 

* 'cnf' if symmetric keys are used, not present if asymmetric keys are used. This parameter is defined in Section 3.2 of {{I-D.ietf-ace-oauth-params}} and contains the symmetric proof-of-possession key that the Client is supposed to use with the KDC.

* 'rs_cnf' if asymmetric keys are used, not present if symmetric keys are used. This parameter is as defined in Section 3.2 of {{I-D.ietf-ace-oauth-params}} and contains information about the public key of the KDC.

* 'exp', contains the lifetime in seconds of the access token. This parameter MAY be omitted if the application defines how the expiration time is communicated to the Client via other means, or if it establishes a default value.

Additionally, the Authorization Response MAY contain the following parameters, which, if included, MUST have the corresponding values:

* 'scope', which mirrors the 'scope' parameter in the Authorization Request (see {{ssec-authorization-request}}). Its value is a CBOR array encoded as a byte string, containing:

  - As first element, the identifier of the specific group or topic the Client is authorized to access.

  - Optionally, as second element, the role (or CBOR array of roles) the Client is authorized to take in the group.

  The encoding of the group or topic identifier and of the role identifiers is application specific.

* Other additional parameters as defined in {{I-D.ietf-ace-oauth-authz}}, if necessary.

<!--
  Jim 14-06: State explicitly what the AS does when receiving an authorization request for a group from a joining node that already has a valid not expired token to join that group (and is probably an actual member). The AS can simply issue a new token.

  TODO: Note that if the joining node valid not expired token (for example if it was previously member of the group), the AS still issue a new token.

-->

The access token MUST contain all the parameters defined above (including the same 'scope' as in this message, if present, or the 'scope' of the Authorization Request otherwise), and additionally other optional parameters the profile requires.

When receiving an Authorization Request from a Client that was previously authorized, and which still owns a valid non expired access token, the AS can simply reply with an Authorization Response including a new access token.

## Token Post

The Client sends a CoAP POST request including the access token to the KDC, as specified in section 5.8.1 of {{I-D.ietf-ace-oauth-authz}}. If the specific ACE profile defines it, the Client MAY use a different endpoint than /authz-info at the KDC to post the access token to. After successful verification, the Client is authorized to receive the group keying material from the KDC and join the group.

Note that this step could be merged with the following message from the Client to the KDC, namely Key Distribution Request.

# Key Distribution {#key-distr}

This section defines how the keying material used for group communication is distributed from the KDC to the Client, when joining the group as a new member.

If not previously established, the Client and the KDC MUST first establish a pairwise secure communication channel using ACE. The exchange of Key Distribution Request-Response MUST occur over that secure channel. The Client and the KDC MAY use that same secure channel to protect further pairwise communications, that MUST be secured.

During this exchange, the Client sends a request to the AS, specifying the group it wishes to join (see {{ssec-key-distribution-request}}). Then, the KDC verifies the access token and that the Client is authorized to join that group; if so, it provides the Client with the keying material to securely communicate with the member of the group (see {{ssec-key-distribution-response}}).

{{fig-key-distr-join}} gives an overview of the exchange described above.

~~~~~~~~~~~
Client                                               KDC
   |                                                  |
   |---- Key Distribution Request: POST /group-id --->|
   |                                                  |
   |<--- Key Distribution Response: 2.01 (Created) ---|
   |                                                  |
~~~~~~~~~~~
{: #fig-key-distr-join title="Message Flow of Key Distribution to a New Group Member" artwork-align="center"}

<!-- Jim 13-07: Should one talk about the ability to use OBSERVE as part of
key distribution?

Marco: It was just briefly mentioned before and not really elaborated. Although it would work, it seems not useful to have it together with a proper rekeying scheme where the KDC takes the initiative anyway. This would result in much more network traffic and epoch-synchronization.
-->

<!-- Jim 13-07: Section 4.x - I am having a hard time trying to figure out the difference between a group and a topic.  The text does not always seem to distinguish these well.

Marco: We could just go for "group", as a collection of devices sharing the same keyign material (i.e. a security group). Then a group can be mapped to a topic of common interest for its members, such as in a pub-sub environment.
-->

The same set of message can also be used for the following cases, when the Client is already a group member:

* The Client wishes to (re-)get the current keying material, for cases such as expiration, loss or suspected mismatch, due to e.g. reboot or missed group rekeying. This is further discussed in {{sec-expiration}}.

* The Client wishes to (re-)get the public keys of other group members, e.g. if it is aware of new nodes joining the group after itself. This is further discussed in {{sec-key-retrieval}}.

Additionally, the format of the payload of the Key Distribution Response ({{ssec-key-distribution-response}}) can be reused for messages sent by the KDC to distribute updated group keying material, in case of a new node joining the group or of a current member leaving the group. The key management scheme used to send such messages could rely on, e.g., multicast in case of a new node joining or unicast in case of a node leaving the group.

<!--
  Jim 14-06: Discuss that a Key Distribution Request/Response can be performed exactly in the same way also by an already member of the group. Mention the cases when this happens, e.g. believed lost of synchronization with the current group security context, crash and reboot and so on, so forced re-synchronization with the correct current security context.

  TODO: Add a general description on when the following msgs are used:
    - join new nodes
    - member for rekeying (triggered by KDC)
    - member after they forgot (crash)
-->

<!-- Jim 13-07: Section 4.x  - cnf - text does not allow for key identifier

Marco: In Section 4.2, we are indicating the key identifier in the optional 'kid' parameter of the COSE Key.
-->

<!-- Jim 13-07: Section X.X - Define a new cnf method to hold the OSCORE context parameters - should it be a normal COSE_Key or something new just to makes sure that it is different.

Marco: Isn't it ok as we are doing with the COSE Key in Section 4.2? Then it works quite fine in ace-oscoap-joining when considering the particular joining of OSCORE groups.
-->

Note that proof-of-possession to bind the access token to the Client is performed by using the proof-of-possession key bound to the access token for establishing secure communication between the Client and the KDC.

## Key Distribution Request {#ssec-key-distribution-request}

The Client sends a Key Distribution request to the KDC.
This corresponds to a CoAP POST request to the endpoint in the KDC associated to the group to join. The endpoint in the KDC is associated to the 'scope' value of the Authorization Request/Response. The payload of this request is a CBOR Map which MAY contain the following fields, which, if included, MUST have the corresponding values:

* 'scope', with value the specific resource that the Client is authorized to access (i.e. group or topic identifier) and role(s), encoded as in {{ssec-authorization-request}}.

<!--
  Jim 14-06: We need an API for distributing all public keys or a specific public key to an endpoint already member of the group. In the first case, the query information can be the identifier of the group/topic. In the second case, the query information can be the endpoint ID associated to the key to be retrieved, as considered as key identifier. This particular details such as "Group ID" and "Sender ID" are specified in the main group OSCORE document as a particular instance of this generic model.

  TODO: define format get_pub_keys: []
-->

* 'get_pub_keys', if the Client wishes to receive the public keys of the other nodes in the group from the KDC. The value is an empty CBOR Array. This parameter may be present if the KDC stores the public keys of the nodes in the group and distributes them to the Client; it is useless to have here if the set of public keys of the members of the group is known in another way, e.g. it was provided by the AS.

<!-- Peter 30-07: get_pub_keys: Instead of empty CBOR array, an empty payload is also possible?

Marco: As a parameter, it must have a type anyway and we said it should be a CBOR array consistently with the usage of this parameter in the following sections.
-->

* 'client_cred', with value the public key or certificate of the Client. If the KDC is managing (collecting from/distributing to the Client) the public keys of the group members, this field contains the public key of the Client.

* 'pub_keys_repos', can be present if a certificate is present in the 'client_cred' field, with value a list of public key repositories storing the certificate of the Client.


## Key Distribution Response {#ssec-key-distribution-response}

<!-- Jim 13-07: Section X.X - Define a new cnf method to hold the OSCORE context parameters - should it be a normal COSE_Key or something new just to makes sure that it is different.

Marco: Isn't it ok as we are doing with the COSE Key here in Section 4.2? Then it works quite fine in ace-oscoap-joining when considering the particular joining of OSCORE groups. Also, OSCORE is ongly a particular case, while this document is general. Also, this phase where keying material is provisinoed is not even ACE anymore, so there is no need to really stick to a 'cnf' parameter.
-->

<!-- Jim 13-07: Question - does somebody talk about doing key derivation for a new kid showing up and by the way where is the gid

Marco: This seems very much related to Group OSCORE, rather than general message format. In fact, it's in oscore-groupcomm that we describe how a new Recipient Context is derived on demand when "a new kid shows up".

Similarly for the Gid, this document keeps a high livel perspective. It's in ace-oscoap-join that we say how the current Group ID is provided to a joining node in the 'serverID' parameter of the COSE Key in the Join Response.
-->

<!-- Jim 13-07: Seciton 4.2 - if you are using profile, then you should return it.

Marco:  Why? This part is not even strictly ACE anymore. Also, the Client knows what kind of response to expect, since it is contacted a specific resource on the KDC in the first place.
-->

The KDC verifies the access token and, if verification succeeds, sends a Key Distribution success Response to the Client. This corresponds to a 2.01 Created message. The payload of this response is a CBOR Map which MUST contain the following fields:

* 'key', used to send the keying material to the Client, as a COSE_Key ({{RFC8152}}) containing the following parameters:
  - 'kty', as defined in {{RFC8152}}.
  - 'k', as defined in {{RFC8152}}.
  - 'exp' (optionally), as defined below. This parameter is RECOMMENDED to be included in the COSE_Key. If omitted, the authorization server SHOULD provide the expiration time via other means or document the default value.
  - 'alg' (optionally), as defined in {{RFC8152}}.
  - 'kid' (optionally), as defined in {{RFC8152}}.
  - 'base iv' (optionally), as defined in {{RFC8152}}.
  - 'clientID' (optionally), as defined in {{I-D.ietf-ace-oscore-profile}}.
  - 'serverID' (optionally), as defined in {{I-D.ietf-ace-oscore-profile}}.
  - 'kdf' (optionally), as defined in {{I-D.ietf-ace-oscore-profile}}.
  - 'slt' (optionally), as defined in {{I-D.ietf-ace-oscore-profile}}.
  - 'cs_alg' (optionally), containing the algorithm value to countersign the message, taken from Table 5 and 6 of {{RFC8152}}.

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

Optionally, the Key Distribution Response MAY contain the following parameters, which, if included, MUST have the corresponding values:

* 'pub\_keys', may only be present if 'get\_pub\_keys' was present in the Key Distribution Request; this parameter is a COSE\_KeySet (see {{RFC8152}}), which contains the public keys of all the members of the group.

* 'group_policies', with value a list of parameters indicating how the group handles specific management aspects. This includes, for instance, approaches to achieve synchronization of sequence numbers among group members. The exact format of this parameter is specific to the profile.

* 'mgt_key_material', with value the administrative keying material to participate in the group rekeying performed by the KDC. The exact format and content depend on the specific rekeying scheme used in the group, which may be specified in the profile.

<!-- Peter 30-07: The parameters group_policies and mgt_key_material are not specified in pubsub-profile draft. Do you ever use them?

Marco: We already use them in the joining draft. Aren't they anyway relevant in the pubsub profile too?
-->

Specific profiles need to specify how exactly the keying material is used to protect the group communication.

If the application requires backward security, the KDC SHALL generate new group keying material and securely distribute it to all the current group members, using the message format defined in this section. Application profiles may define alternative message formats.

TBD: define for verification failure

# Removal of a Node from the Group {#sec-node-removal}

This section describes at a high level how a node can be removed from the group.

If the application requires forward security, the KDC SHALL generate new group keying material and securely distribute it to all the current group members but the leaving node, using the message format defined in {{ssec-key-distribution-response}}. Application profiles may define alternative message formats.

## Expired Authorization

If the node is not authorized anymore, the AS can directly communicate that to the KDC. Alternatively, the access token might have expired. If Token introspection is provided by the AS, the KDC can use it as per Section 5.7 of {{I-D.ietf-ace-oauth-authz}}, in order to verify that the access token is still valid.

Either case, once aware that a node is not authorized anymore, the KDC has to remove the unauthorized node from the list of group members, if the KDC keeps track of that.

## Request to Leave the Group

A node can actively request to leave the group. In this case, the Client can send a request formatted as follows to the KDC, to abandon the group.

TBD: Format of the message to leave the group

<!-- Jim 13-07: Section 5.2 - What is the message to leave - can I leave one scope but not another?  Can I just give up a role?

Marco: We should define an actual message, like the ones for retrieving updating keying material in Section 6. It can be like the one in Section 6.1, only with the second part of 'scope' present and encoded as an empty CBOR array.

Marco: 'scope' encodes one group and some roles. So a node is supposed to leave that group altogether, with all its roles. If the node wants to stay in the group with less roles, it is just fine that is stops playing the roles it is not interested in anymore.
-->

The KDC should then remove the leaving node from the list of group members, if the KDC keeps track of that.

Note that, after having left the group, a node may wish to join it again. Then, as long as the node is still authorized to join the group, i.e. it has a still valid access token, it can re-request to join the group directly to the KDC without needing to retrieve a new access token from the AS. This means that the KDC needs to keep track of nodes with valid access tokens, before deleting all information about the leaving node.

# Retrieval of Updated Keying Material {#sec-expiration}

A node stops using the group keying material upon its expiration, according to the 'exp' parameter specified in the retained COSE Key. Then, if it wants to continue participating in the group communication, the node has to request new updated keying material to the KDC.

The Client may perform the same request to the KDC also upon receiving messages from other group members without being able to correctly decrypt them. This may be due to a previous update of the group keying material (rekeying) triggered by the KDC, that the Client was not able to receive or decrypt.


<!-- Jim 13-07: Comment somewhere about getting strike zones setup correctly for a newly seen sender of messages. Ptr to OSCORE?

Marco: Just expanded the second paragraph in Section 6.

If this is about details on deriving Recipient Contexts, that's OSCORE specific and should not be here.

If this is about retrieving the public key of a newly joined sender, that's actually a general requirement and is not strictly related to OSCORE.

Is there any other convenient OSCORE thing which is reusable here and we are missing?
-->


Note that policies can be set up so that the Client sends a request to the KDC only after a given number of unsuccessfully decrypted incoming messages.

## Key Re-Distribution Request

To request a re-distribution of keying material, the Client sends a shortened Key Distribution Request to the KDC ({{ssec-key-distribution-request}}), formatted as follows. The payload MUST contain only the following field:

* 'scope', which contains only the identifier of the specific group or topic, encoded as in {{ssec-authorization-request}}. That is, the role field is not present.

<!-- In some cases, it is not necessary to include the scope parameter, for instance if the KDC maintains a list of active group members for each managed group, and the Client is member of only one group. The Client MUST include the scope parameter if it is a member of multiple groups under the same KDC.


 Peter 30-07: "In some cases ... same KDC". Suggest to remove. In a fast changing environment, this may lead to many error messages if not wrong behavior; Imagine group GA is the only group. C is member of GA. GA is removed and GB is entered as the only group. C wants to leave/join GA, and accesses GB.

Marco: It makes sense, should we then just make 'scope' mandatory?
-->

## Key Re-Distribution Response

The KDC receiving a Key Re-Distribution Request MUST check that it is storing a valid access token from that client for that scope.

TODO: defines error response if it does not have it / is not valid.

The KDC replies to the Client with a Key Distribution Response containing the 'key' parameter, and optionally 'group_policies' and 'mgt_key_material', as specified in {{ssec-key-distribution-response}}. Note that this response might simply re-provide the same keying material currently owned by the Client, if it has not been renewed.

# Retrieval of Public Keys for Group Members {#sec-key-retrieval}

In case the KDC maintains the public keys of group members, a node in the group can contact the KDC to request public keys of either all group members or a specified subset, using the messages defined below.

{{fig-public-key-req-resp}} gives an overview of the exchange described above.

~~~~~~~~~~~
Client                                         KDC
   |                                            |
   |---- Public Key Request: POST /group-id --->|
   |                                            |
   |<--- Public Key Response: 2.01 (Created) ---|
   |                                            |
~~~~~~~~~~~
{: #fig-public-key-req-resp title="Message Flow of Public Key Request-Response" artwork-align="center"}

Note that these messages can be combined with the Key Re-Distribution messages in {{sec-expiration}}, to request at the same time the keying material and the public keys. In this case, either a new endpoint at the KDC may be used, or additional information needs to be sent in the request payload, to distinguish these combined messages from the Public Key messages described below, since they would be identical otherwise.

## Public Key Request

To request public keys, the Client sends a shortened Key Distribution Request to the KDC ({{ssec-key-distribution-request}}), formatted as follows. The payload of this request MUST contain the following fields:

* 'get_pub_keys', which has as value a CBOR array including either:
  - no elements, i.e. an empty array, in order to request the public key of all current group members; or
  - N elements, each of which is the identifier of a group member, in order to request the public key of the specified nodes.

* 'scope', which contains only the identifier of the specific group or topic, encoded as in {{ssec-authorization-request}}. That is, the role field is not present.

<!-- In some cases, it is not necessary to include the scope parameter, for instance if the KDC maintains a list of active group members for each managed group, and the Client is member of only one group. The Client MUST include the scope parameter if it is a member of multiple groups under the same KDC.


 Peter 30-07: "In some cases ... same KDC". Suggest to remove. In a fast changing environment, this may lead to many error messages if not wrong behavior; Imagine group GA is the only group. C is member of GA. GA is removed and GB is entered as the only group. C wants to leave/join GA, and accesses GB.

Marco: It makes sense, should we then just make 'scope' mandatory?
-->

## Public Key Response

The KDC replies to the Client with a Key Distribution Response containing only the 'pub_keys' parameter, as specified in {{ssec-key-distribution-response}}. The payload of this response contains the following field:

* 'pub_keys', which contains either:

  - the public keys of all the members of the group, if the 'get_pub_keys' parameter of the Public Key request was an empty array; or

  - the public keys of the group members with the identifiers specified in the 'get_pub_keys' parameter of the Public Key request.

The KDC ignores possible identifiers included in the 'get_pub_keys' parameter of the Public Key request if they are not associated to any current group member.

# Security Considerations

The KDC must renew the group keying material upon its expiration.

The KDC should renew the keying material upon group membership change, and should provide it to the current group members through the rekeying scheme used in the group.

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

The following individuals were helpful in shaping this document: Ben Kaduk, John Mattsson, Jim Schaad, Ludwig Seitz, Göran Selander and Peter van der Stok.

The work on this document has been partly supported by the EIT-Digital High Impact Initiative ACTIVE.

--- fluff
