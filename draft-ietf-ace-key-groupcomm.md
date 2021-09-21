---
title: Key Provisioning for Group Communication using ACE
abbrev: Key Provisioning for Group Communication
docname: draft-ietf-ace-key-groupcomm-latest

ipr: trust200902
wg: ACE Working Group
cat: std

coding: utf-8
pi:
  toc: yes
  sortrefs: yes
  symrefs: yes
  tocdepth: 4

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
  RFC6749:
  RFC6838:
  RFC8126:
  RFC8174:
  RFC8610:
  RFC7252:
  RFC7967:
  RFC8742:
  RFC8747:
  RFC8949:
  I-D.ietf-cose-rfc8152bis-struct:
  I-D.ietf-cose-rfc8152bis-algs:
  I-D.ietf-ace-oauth-authz:
  I-D.ietf-core-oscore-groupcomm:
  I-D.ietf-ace-aif:
  COSE.Algorithms:
    author:
      org: IANA
    date: false
    title: COSE Algorithms
    target: https://www.iana.org/assignments/cose/cose.xhtml#algorithms
  COSE.Header.Parameters:
    author:
      org: IANA
    date: false
    title: COSE Header Parameters
    target: https://www.iana.org/assignments/cose/cose.xhtml#header-parameters
    
informative:

  RFC7519:
  RFC2093:
  RFC2094:
  RFC2627:
  RFC7959:
  RFC8259:
  RFC8613:
  RFC8392:
  RFC7641:
  I-D.ietf-core-groupcomm-bis:
  I-D.ietf-core-coap-pubsub:
  I-D.ietf-ace-oscore-profile:
  I-D.ietf-ace-dtls-authorize:
  I-D.ietf-ace-mqtt-tls-profile:


--- abstract

This document defines message formats and procedures for requesting and distributing group keying material using the ACE framework, to protect communications between group members.

--- middle

# Introduction {#intro}

This document expands the ACE framework {{I-D.ietf-ace-oauth-authz}} to define the message exchanges used to request, distribute and renew the keying material in a group communication scenario, e.g., based on multicast {{I-D.ietf-core-groupcomm-bis}} or on publish-subscribe messaging {{I-D.ietf-core-coap-pubsub}}.
The ACE framework is based on CBOR {{RFC8949}}, so CBOR is the format used in this specification. However, using JSON {{RFC8259}} instead of CBOR is possible, using the conversion method specified in {{Sections 6.1 and 6.2 of RFC8949}}.

Profiles that use group communication can build on this document, by defining a number of details such as the exact group communication protocol and security protocols used. The specific list of details a profile needs to define is shown in {{req}}.

If the application requires backward and forward security, new keying material is generated and distributed to the group upon membership changes. A key management scheme performs the actual distribution of the new keying material to the group. In particular, the key management scheme rekeys the current group members when a new node joins the group, and the remaining group members when a node leaves the group. Rekeying mechanisms can be based on {{RFC2093}}, {{RFC2094}} and {{RFC2627}}.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Readers are expected to be familiar with the terms and concepts described in  {{I-D.ietf-ace-oauth-authz}}{{I-D.ietf-cose-rfc8152bis-struct}}{{I-D.ietf-cose-rfc8152bis-algs}}, such as Authorization Server (AS) and Resource Server (RS).

This document uses names or identifiers for groups and nodes. Their different meanings are summarized here:

* "Group name" is the invariant once established identifier of the group. It is used in the communication between AS, RS and Client to identify the group.

* "GROUPNAME" is the invariant once established text string used in URIs. GROUPNAME maps to the group name of a group, although it is not necessarily the same.

* "Group identifier" is the identifier of the group keying material. Opposite to group name and GROUPNAME, this identifier changes over time, when the keying material is updated.

* "Node name" is the invariant once established identifier of the node. It is used in the communication between AS, RS and Client to identify a member of the group.

* "NODENAME" is the invariant once established text string used in URIs. NODENAME is used to identify a node in a group.

This document additionally uses the following terminology:

* Transport profile, to indicate a profile of ACE as per {{Section 5.8.4.3 of I-D.ietf-ace-oauth-authz}}. A transport profile specifies the communication protocol and communication security protocol between an ACE Client and Resource Server, as well as proof-of-possession methods, if it supports proof-of-possession access tokens, etc. Transport profiles of ACE include, for instance, {{I-D.ietf-ace-oscore-profile}}, {{I-D.ietf-ace-dtls-authorize}} and {{I-D.ietf-ace-mqtt-tls-profile}}.

* Application profile, that defines how applications enforce and use supporting security services they require. These services may include, for instance, provisioning, revocation and distribution of keying material. An application profile may define specific procedures and message formats.

# Overview

The full procedure can be separated in two phases: the first one follows the ACE framework, between Client, AS and KDC; the second one is the key distribution between Client and KDC. After the two phases are completed, the Client is able to participate in the group communication, via a Dispatcher entity.

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
|            |<-------->|            |<------->||   Group   |
+------------+          +------------+         +|  members  |
                                                +-----------+
~~~~~~~~~~~
{: #fig-roles title="Key Distribution Participants" artwork-align="center"}

The following participants (see {{fig-roles}}) take part in the authorization and key distribution.

* Client (C): node that wants to join the group communication. It can request write and/or read rights.

* Authorization Server (AS): same as AS in the ACE Framework; it enforces access policies, and knows if a node is allowed to join a given group with write and/or read rights.

* Key Distribution Center (KDC): maintains the keying material to protect group communications, and provides it to Clients authorized to join a given group. During the first part of the exchange ({{sec-auth}}), it takes the role of the RS in the ACE Framework. During the second part ({{key-distr}}), which is not based on the ACE Framework, it distributes the keying material. In addition, it provides the latest keying material to group members when requested or, if required by the application, when membership changes.

* Dispatcher: entity through which the Clients communicate with the group and which distributes messages to the group members. Examples of dispatchers are: the Broker node in a pub-sub setting; a relayer node for group communication that delivers group messages as multiple unicast messages to all group members; an implicit entity as in a multicast communication setting, where messages are transmitted to a multicast IP address and delivered on the transport channel.

This document specifies a mechanism for:

* Authorizing a new node to join the group ({{sec-auth}}), and providing it with the group keying material to communicate with the other group members ({{key-distr}}).

* Allowing a group member to retrieve group keying material ({{update-keys}} and {{new-keys}}).

* Allowing a group member to retrieve public keys of other group members ({{sec-key-retrieval}}) and to provide an updated public key ({{update-pub-key}}).

* Allowing a group member to leave the group ({{sec-node-removal}}).

* Evicting a group member from the group ({{sec-node-removal}}).

* Renewing and re-distributing the group keying material (rekeying) upon a membership change in the group ({{ssec-group-leaving}} and {{sec-node-removal}}).

{{fig-flow}} provides a high level overview of the message flow for a node joining a group. The message flow can be expanded as follows.

1. The joining node requests an access token from the AS, in order to access one or more group-membership resources at the KDC and hence join the associated groups.

   This exchange between Client and AS MUST be secured, as specified by the transport profile of ACE used between Client and KDC. Based on the response from the AS, the joining node will establish or continue using a secure communication association with the KDC.

2. The joining node transfers authentication and authorization information to the KDC, by transferring the obtained access token. This is typically achieved by including the access token in a request sent to the /authz-info endpoint at the KDC.

   Once this exchange is completed, the joining node MUST have a secure communication association established with the KDC, before joining a group under that KDC.

   This exchange and the following secure communications between the Client and the KDC MUST occur in accordance with the transport profile of ACE used between Client and KDC, such as the DTLS transport profile {{I-D.ietf-ace-dtls-authorize}} and OSCORE transport profile {{I-D.ietf-ace-oscore-profile}} of ACE.

3. The joining node starts the joining process to become a member of the group, by sending a request to the related group-membership resource at the KDC. At the end of the joining process, the joining node has received from the KDC the parameters and keying material to securely communicate with the other members of the group. Also, the KDC has stored the association between the authorization information from the access token and the secure session with the joining node.

4. The joining node and the KDC maintain the secure association, to support possible future communications. These especially include key management operations, such as retrieval of updated keying material or participation to a group rekeying process.

5. The joining node can communicate securely with the other group members, using the keying material provided in step 3.


~~~~~~~~~~~
        C                             AS   KDC                 Group
        |                              |   |                  Members
      / |                              |   |                      |
     |  |--- Authorization Request --->|   |                      |
     |  |                              |   |                      |
     |  |<-- Authorization Response ---|   |                      |
(*) <   |                              |   |                      |
     |  |                              |   |                      |
     |  |---  Token Transfer Request ----->|                      |
     |  |                                  |                      |
     |  |<--- Token Transfer Response -----|                      |          
      \ |                              |   |                      |
        |                              |   |                      |
        |------ Joining Request ------>|   |                      |
        |                              |   |                      |
        |<----- Joining Response ------|   | -- Group Rekeying -->|
        |                              |   |                      |
        |                              |   |                      |
        |                                        Dispatcher       |
        |                                            |            |
        |<======= Secure group communication ========|===========>|
        |                                            |            |
            
(*) Defined in the ACE framework
~~~~~~~~~~~
{: #fig-flow title="Message Flow Upon New Node's Joining" artwork-align="center"}

# Authorization to Join a Group {#sec-auth}

This section describes in detail the format of messages exchanged by the participants when a node requests access to a given group. This exchange is based on ACE {{I-D.ietf-ace-oauth-authz}}.

As defined in {{I-D.ietf-ace-oauth-authz}}, the Client requests the AS for the authorization to join the group through the KDC (see {{ssec-authorization-request}}). If the request is approved and authorization is granted, the AS provides the Client with a proof-of-possession access token and parameters to securely communicate with the KDC (see {{ssec-authorization-response}}).

Communications between the Client and the AS MUST be secured, according to what is defined by the used transport profile of ACE. The Content-Format used in the message also depends on the used transport profile of ACE. For example, it can be application/ace+cbor for the first two messages and application/cwt for the third message, which are defined in the ACE framework.

The transport profile of ACE also defines a number of details such as the communication and security protocols used with the KDC (see Appendix C of {{I-D.ietf-ace-oauth-authz}}).

{{fig-group-member-registration}} gives an overview of the exchange described above.

~~~~~~~~~~~
Client                                             AS    KDC
   |                                                |     |
   |---- Authorization Request: POST /token ------->|     |
   |                                                |     |
   |<--- Authorization Response: 2.01 (Created) ----|     |
   |                                                |     |
   |---- Token Transfer Request: POST /authz-info ------->|
   |                                                |     |
   |<--- Token Transfer Response: 2.01 (Created) -------->|
   |                                                |     |
~~~~~~~~~~~
{: #fig-group-member-registration title="Message Flow of Join Authorization" artwork-align="center"}

## Authorization Request {#ssec-authorization-request}

The Authorization Request sent from the Client to the AS is defined in {{Section 5.8.1 of I-D.ietf-ace-oauth-authz}} and MAY contain the following parameters, which, if included, MUST have the corresponding values:

* 'scope', containing the identifier of the specific groups, or topics in the case of pub-sub, that the Client wishes to access, and optionally the roles that the Client wishes to take.

   This value is a CBOR byte string, wrapping a CBOR array of one or more  entries.

   By default, each entry is encoded as specified by {{I-D.ietf-ace-aif}}. The object identifier "Toid" corresponds to the group name and MUST be encoded as a tstr. The permission set "Tperm" indicates the roles that the client wishes to take in the group. It is up to the application profiles to define "Tperm" (REQ2) and register "Toid" and "Tperm" to fit the use case. An example of scope using the AIF format is given in {{cddl-ex-0}}.
   
   Otherwise, each scope entry can be defined as a CBOR array, which contains:

  - As first element, the identifier of the specific group or topic, encoded as a tstr.

  - Optionally, as second element, the role (or CBOR array of roles) that the Client wishes to take in the group. This element is optional since roles may have been pre-assigned to the Client, as associated to its verifiable identity credentials. Alternatively, the application may have defined a single, well-known role for the target resource(s) and audience(s).

  In each entry, the encoding of the role identifiers is application specific, and part of the requirements for the application profile (REQ2).
  In particular, the application profile may specify CBOR values to use for abbreviating role identifiers (OPT7).
  
  {{cddl-ex}} provides an example of CDDL definition {{RFC8610}} of scope using the format above, with group name and role identifiers encoded as text strings.

* 'audience', with an identifier of a KDC.

As defined in {{I-D.ietf-ace-oauth-authz}}, other additional parameters can be included if necessary.

~~~~~~~~~~~~~~~~~~~~ CDDL
gname = tstr

permissions = uint . bits roles

roles = &(
   Requester: 1,
   Responder: 2,
   Monitor: 3,
   Verifier: 4
)

scope_entry = AIF_Generic<gname, permissions>

scope = << [ + scope_entry ] >>
~~~~~~~~~~~~~~~~~~~~
{: #cddl-ex-0 title="Example CDLL definition of scope, using the default Authorization Information Format"}

~~~~~~~~~~~~~~~~~~~~ CDDL
gname = tstr

role = tstr

scope_entry = [ gname , ? ( role / [ 2*role ] ) ]

scope = << [ + scope_entry ] >>
~~~~~~~~~~~~~~~~~~~~
{: #cddl-ex title="CDLL definition of scope, using as example group name encoded as tstr and role as tstr"}

## Authorization Response {#ssec-authorization-response}

The Authorization Response sent from the AS to the Client is defined in {{Section 5.8.2 of I-D.ietf-ace-oauth-authz}}. Note that the parameter 'expires_in' MAY be omitted if the application defines how the expiration time is communicated to the Client via other means, or if it establishes a default value.

Additionally, when included, the following parameter MUST have the corresponding values:

* 'scope' has the same format and encoding of 'scope' in the Authorization Request, defined in {{ssec-authorization-request}}. If this parameter is not present, the granted scope is equal to the one requested in {{ssec-authorization-request}}.

The proof-of-possession access token (in 'access_token' above) MUST contain the following parameters:

* a confirmation claim (see for example 'cnf' defined in {{Section 3.1 of RFC8747}} for CWT);

* an expiration time claim (see for example 'exp' defined in {{Section 3.1.4 of RFC8392}} for CWT);

* a scope claim (see for example 'scope' registered in {{Section 8.14 of I-D.ietf-ace-oauth-authz}} for CWT).

   This claim specifies the same access control information as in the 'scope' parameter of the Authorization Response, if the parameter is present in the message, or as in the 'scope' parameter of the Authorization Request otherwise.

   By default, this claim has the same encoding as the 'scope' parameter in the Authorization Request, defined in {{ssec-authorization-request}}.

   Optionally, an alternative extended format of scope defined in {{sec-extended-scope}} can be used. This format explicitly signals the semantics used to express the actual access control information, and according to which this has to be parsed. This enables a Resource Server to correctly process a received access token, also in case:
   
     - The Resource Server implements a KDC that supports multiple application profiles of this specification, using different scope semantics; and/or
   
     - The Resource Server implements further services beyond a KDC for group communication, using different scope semantics.

    If the Authorization Server is aware that this applies to the Resource Server for which the access token is issued, the Authorization Server SHOULD use the extended format of scope defined in {{sec-extended-scope}}.
    
The access token MAY additionally contain other claims that the transport profile of ACE requires, or other optional parameters.

When receiving an Authorization Request from a Client that was previously authorized, and for which the AS still owns a valid non-expired access token, the AS MAY reply with that token. Note that it is up to application profiles of ACE to make sure that re-posting the same token does not cause re-use of keying material between nodes (for example, that is done with the use of random nonces in {{I-D.ietf-ace-oscore-profile}}).

## Token Transferring {#token-post}

The Client sends a Token Transfer Request to the KDC, i.e., a CoAP POST request including the access token and targeting the authz-info endpoint, as specified in {{Section 5.8.1 of I-D.ietf-ace-oauth-authz}}.

This request differs from the one defined in {{I-D.ietf-ace-oauth-authz}}, since it allows to transport additional information about the public keys used in the group to ensure source authentication, as well as possible additional group parameters.

The joining node MAY ask for this information from the KDC through the same Token Transfer Request. In this case, the message MUST have Content-Format set to application/ace+cbor defined in {{Section 8.16 of I-D.ietf-ace-oauth-authz}}, and the message payload MUST be formatted as a CBOR map, which MUST include the access token. The CBOR map MAY additionally include the following parameter, which, if included, MUST have the corresponding values:

* 'sign_info' defined in {{sign-info}}, specifying the CBOR simple value 'null' (0xf6) to request information about the signature algorithm, signature algorithm parameters, signature key parameters and about the exact encoding of public keys used in the groups that the client has been authorized to join.

Alternatively, the joining node may retrieve this information by other means.

After successful verification, the Client is authorized to receive the group keying material from the KDC and join the group. Hence, the KDC replies to the Client with a Token Transfer Response, i.e., a  CoAP 2.01 (Created) response.

The Token Transfer Response MUST have Content-Format "application/ace+cbor", and its payload is a CBOR map. Note that this deviates from what is defined in the ACE framework, where the response from the authz-info endpoint is defined as conveying no payload (see {{Section 5.10.1 of I-D.ietf-ace-oauth-authz}}).

If the access token contains a role that requires the Client to send its own public key to the KDC when joining the group, the CBOR map MUST include the parameter 'kdcchallenge' defined in {{kdcchallenge}}, specifying a dedicated challenge N_S generated by the KDC. The Client uses this challenge to prove possession of its own private key (see the 'client_cred_verify' parameter in {{key-distr}}).

The KDC MUST store the 'kdcchallenge' value associated to the Client at least until it receives a join request from it (see {{ssec-key-distribution-exchange}}), to be able to verify that the Client possesses its own private key. The same challenge MAY be reused several times by the Client, to generate a new proof of possession, e.g., in case of update of the public key, or to join a different group with a different signing key. Therefore, it is RECOMMENDED that the KDC keeps storing the 'kdcchallenge' after the first join is processed as well. If the KDC has already discarded the 'kdcchallenge', that will trigger an error response with a newly generated 'kdcchallenge' that the Client can use to restart the join process, as specified in {{ssec-key-distribution-exchange}}.

If 'sign_info' is included in the Token Transfer Request, the KDC MAY include the 'sign_info' parameter in the Token Transfer Response, as per the format defined in {{sign-info}}. Note that the field 'id' specifies the name, or array of group names, for which the 'sign_info_entry' applies to.

Note that the CBOR map specified as payload of the 2.01 (Created) response may include further parameters, e.g., according to the used transport profile of ACE. Application profiles of this specification MAY define additional parameters to use within this exchange (OPT2).

Application profiles of this specification MAY define alternative specific negotiations of parameter values for the signature algorithm and signature keys, if 'sign_info' is not used (OPT1).

If allowed by the used transport profile of ACE, the Client may provide the Access Token to the KDC by other means than the Token Transfer Request. An example is the DTLS transport profile of ACE, where the Client can provide the access token to the KDC during the secure session establishment (see {{Section 3.3.2 of I-D.ietf-ace-dtls-authorize}}).

### 'sign_info' Parameter {#sign-info}

The 'sign_info' parameter is an OPTIONAL parameter of the request and response messages exchanged between the Client and the authz-info endpoint at the RS (see {{Section 5.10.1. of I-D.ietf-ace-oauth-authz}}).

This parameter allows the Client and the RS to exchange information about a signature algorithm and about public keys to accordingly use for signature verification. Its exact semantics and content are application specific.

In this specification and in application profiles building on it, this parameter is used to exchange information about the signature algorithm and about public keys to be used with it, in the groups indicated by the transferred acces token as per its 'scope' claim (see {{ssec-authorization-response}}).

When used in the Token Transfer Request sent to the KDC (see {{token-post}}), the 'sign_info' parameter specifies the CBOR simple value 'null' (0xf6). This is done to ask for information about the signature algorithm and about the public keys used in the groups that the Client has been authorized to join - or to have a more restricted interaction as per its granted roles (e.g., the Client is an external signature verifier).

When used in the following Token Transfer Response from the KDC (see {{token-post}}), the 'sign_info' parameter is a CBOR array of one or more elements. The number of elements is at most the number of groups that the client has been authorized to join - or to have a more restricted interaction (see above). Each element contains information about signing parameters and about public keys for one or more groups, and is formatted as follows.

* The first element 'id' is a group name or an array of group names, associated to groups for which the next four elements apply. In the following, each specified group name is referred to as 'gname'.

* The second element 'sign_alg' is an integer or a text string if the POST request included the 'sign_info' parameter with value the CBOR simple value 'null' (0xf6), and indicates the signature algorithm used in the groups identified by the 'gname' values. It is REQUIRED of the application profiles to define specific values that this parameter can take (REQ3), selected from the set of signing algorithms of the COSE Algorithms registry {{COSE.Algorithms}}.

* The third element 'sign_parameters' is a CBOR array indicating the parameters of the signature algorithm used in the groups identified by the 'gname' values. Its content depends on the value of 'sign_alg'. It is REQUIRED of the application profiles to define the possible values and structure for the elements of this parameter (REQ4).

* The fourth element 'sign_key_parameters' is a CBOR array indicating the parameters of the key used with the signature algorithm, in the groups identified by the 'gname' values. Its content depends on the value of 'sign_alg'. It is REQUIRED of the application profiles to define the possible values and structure for the elements of this parameter (REQ5).

* The fifth element 'pub_key_enc' parameter is either a CBOR integer indicating the encoding of public keys used in the groups identified by the 'gname' values, or has value the CBOR simple value 'null' (0xf6) indicating that the KDC does not act as repository of public keys for group members. Its acceptable integer values are taken from the 'Label' column of the "COSE Header Parameters" Registry {{COSE.Header.Parameters}}. It is REQUIRED of the application profiles to define specific values to use for this parameter, consistently with the acceptable formats of public keys (REQ6).

The CDDL notation {{RFC8610}} of the 'sign_info' parameter is given below.

~~~~~~~~~~~ CDDL
sign_info = sign_info_req / sign_info_resp

sign_info_req  = nil                   ; in the Token Transfer
                                       ; Request to the KDC
   
sign_info_resp = [ + sign_info_entry ] ; in the Token Transfer
                                       ; Response from the KDC

sign_info_entry =
[
  id : gname / [ + gname ],
  sign_alg : int / tstr,
  sign_parameters : [ any ],
  sign_key_parameters : [ any ],
  pub_key_enc = int / nil
]

gname = tstr
~~~~~~~~~~~

This format is consistent with every signature algorithm currently defined in {{I-D.ietf-cose-rfc8152bis-algs}}, i.e., with algorithms that have only the COSE key type as their COSE capability. {{sec-future-cose-algs}} describes how the format of each 'sign_info_entry' can be generalized for possible future registered algorithms having a different set of COSE capabilities.

### 'kdcchallenge' Parameter {#kdcchallenge}

The 'kdcchallenge' parameter is an OPTIONAL parameter of response message returned from the authz-info endpoint at the RS, as defined in {{Section 5.10.1 of I-D.ietf-ace-oauth-authz}}. This parameter contains a challenge generated by the RS and provided to the Client.

In this specification and in application profiles building on it, the Client may use this challenge to prove possession of its own private key in the Joining Request (see the 'client_cred_verify' parameter in {{gid-post}}).

# KDC Functionalities {#key-distr}

This section describes the functionalities provided by the KDC, as related to the provisioning of the keying material as well as to the group membership management.

In particular, this section defines the interface available at the KDC; specifies the handlers of each resource provided by the KDC interface; and describes how Clients interact with those resources to join a group and to perform additional operations as group members.

As most important operation after trasferring the access token to the KDC, the Client can perform a "Joining" exchange with the KDC, by specifying the group it wishes to join (see {{ssec-key-distribution-exchange}}). Then, the KDC verifies the access token and that the Client is authorized to join the specified group. If so, the KDC provides the Client with the keying material to securely communicate with the other members of the group.

Later on as a group member, the Client can also rely on the interface at the KDC to perform additional operations, consistently with the roles it has in the group.

## Interface at the KDC {#kdc-if}

The KDC provides its interface by hosting the following resources. Note that the root url-path "ace-group" used hereafter is a default name; implementations are not required to use this name, and can define their own instead. The Interface Description (if=) Link Target Attribute value "ace.group" is registered in {{if-ace-group}} and can be used to describe this interface.

If request messages sent to the KDC as well as success response messages from the KDC include a payload and specify a Content-Format, those messages MUST have Content-Format set to application/ace-groupcomm+cbor, defined in {{content-type}}.

* /ace-group : this resource is invariant once established, and indicates that this specification is used. If other applications run on a KDC implementing this specification and use this same resource, those applications will collide, and a mechanism will be needed to differentiate the endpoints.

  A Client can interact with this resource to retrieve a set of group names, each corresponding to one of the specified group identifiers. This operation is described in {{retrieval-gnames}}.

* /ace-group/GROUPNAME : one such sub-resource to /ace-group is hosted for each group with name GROUPNAME that the KDC manages, and contains the symmetric group keying material for that group.
  
  A Client can interact with this resource in order to join the group with name GROUPNAME, or later as a group member to retrieve the current group keying material. These operations are described in {{ssec-key-distribution-exchange}} and {{ssec-key-material-retrieval}}, respectively.
  
  If the value of the GROUPNAME URI path and the group name in the access token scope ('gname' in {{ssec-authorization-response}}) do not match, the KDC MUST implement a mechanism to map the GROUPNAME value in the URI to the group name, in order to refer to the correct group (REQ1).

* /ace-group/GROUPNAME/pub-key : this resource is invariant once established, and contains the public keys of all the members of the group with name GROUPNAME.
  
  A Client can interact with this resource in order to get the public keys of other group members, i.e., all of them or a subset of them by specifying filter criteria. These operations are described in {{sec-key-retrieval-all}} and {{sec-key-retrieval}}, respectively.

* /ace-group/GROUPNAME/policies : this resource is invariant once established, and contains the group policies of the group with name GROUPNAME.
  
  A Client can interact with this resource in order to retrieve the group policies. This operation is described in {{policies}}.

* /ace-group/GROUPNAME/num : this resource is invariant once established, and contains the current version number for the symmetric group keying material of the group with name GROUPNAME.

  A Client can interact with this resource in order to retrieve the version number of the keying material currently used in the group. This operation is described in {{key-version}}.
  
* /ace-group/GROUPNAME/nodes/NODENAME : one such sub-resource of /ace-group/GROUPNAME is hosted for each group member of the group with name GROUPNAME. Each of such resources is identified by the node name NODENAME of the associated group member, and contains the group keying material and the individual keying material for that group member.

  A Client can interact with this resource in order to retrieve the current group keying material together with its the individual keying material; request new individual keying material to use in the group; and leave the group. These operations are described in {{update-keys}}, {{new-keys}}, and {{ssec-group-leaving}}, respectively.

* /ace-group/GROUPNAME/nodes/NODENAME/pub-key : this resource is invariant once established, and contains the individual public keying material for the node with name NODENAME, as group member of the group with name GROUPNAME.

  A Client can interact with this resource in order to upload at the KDC a new public key to use in the group. This operation is described in {{update-pub-key}}.

  This resource is not created if the group member does not have individual public keying material to use in the group, or if the KDC does not store the public keys of group members. 
  
The KDC is expected to fully provide the interface defined above. It is otherwise REQUIRED of the application profiles of this specification to indicate which resources are not hosted, i.e., which parts of the KDC interface defined in this section are not supported (REQ25). Application profiles of this specification MAY extend the KDC interface, by defining additional resources and their handlers.

It is REQUIRED of the application profiles of this specification to register a Resource Type for the root url-path (REQ7). This Resource Type can be used to discover the correct url to access at the KDC. This Resource Type can also be used at the GROUPNAME sub-resource, to indicate different application profiles for different groups.

It is REQUIRED of the application profiles of this specification to define what operations (e.g., CoAP methods) are allowed on each resource, for each role defined in {{ssec-authorization-request}} according to REQ2 (REQ8).

### Error Handling {#kdc-if-errors}

Upon receiving a request from a Client, the KDC MUST check that it is storing a valid access token from that Client. In case the KDC does not store any such valid access token, the KDC MUST respond to the Client with a 4.01 (Unauthorized) error message.

Unless the request targets the /ace-group resource, the KDC MUST especially check that it is storing a valid access token from that Client for the group with name GROUPNAME associated to the endpoint. In case the KDC stores a valid access token but this does not allow the client to access the group associated to the endpoint, the KDC MUST respond to the Client with a 4.03 (Forbidden) error message.

Some error responses from the KDC can have Content-Format set to application/ace-groupcomm+cbor. In such a case, the paylod of the response MUST be a CBOR map, which includes the following fields.

* 'error', with value a CBOR integer specifying the error occurred at the KDC. The value is taken from the "Value" column of the "ACE Groupcomm Errors" registry defined in {{iana-ace-groupcomm-errors}} of this specification. This field MUST be present.

* 'error_description', with value a CBOR text string specifying a human-readable description of the error occurred at the KDC. This field MAY be present.

CBOR labels for the 'error' and 'error_description' fields are defined in {{params}}.

{{error-types}} of this specification defines an initial set of error identifiers, as possible values for the 'error' field. Application profiles of this specification MAY define additional value (OPT11).

## /ace-group

This resource implements the FETCH handler.

### FETCH Handler {#ace-group-fetch}

The FETCH handler receives group identifiers and returns the corresponding group names and GROUPNAME URIs.

The handler expects a request with payload formatted as a CBOR map, which MUST contain the following fields:

* 'gid', whose value is encoded as a CBOR array, containing one or more group identifiers. The exact encoding of group identifier MUST be specified by the application profile (REQ9). The Client indicates that it wishes to receive the group names and GROUPNAMEs of all groups having these identifiers.

The handler identifies the groups that are secured by the keying material identified by those group identifiers.

Then, the handler returns a 2.05 (Content) message response with payload formatted as a CBOR map that MUST contain the following fields:

* 'gid', whose value is encoded as a CBOR array, containing zero or more group identifiers. The handler indicates that those are the identifiers it is sending group names and GROUPNAMEs for. This CBOR array is a subset of the 'gid' array in the FETCH request.

* 'gname', whose value is encoded as a CBOR array, containing zero or more group names. The elements of this array are encoded as text strings. Each element of index i of this CBOR array corresponds to the element of group identifier i in the 'gid' array.

* 'guri', whose value is encoded as a CBOR array, containing zero or more URIs, each indicating a GROUPNAME resource.  The elements of this array are encoded as text strings. Each element of index i of this CBOR array corresponds to the element of group identifier i in the 'gid' array.

If the KDC does not find any group associated to the specified group identifiers, the handler returns a response with payload formatted as a CBOR byte string of zero length.

Note that the KDC only verifies that the node is authorized by the AS to access this resource. Nodes that are not members of the group but are authorized to do signature verification on the group messages may be allowed to access this resource, if the application needs it.

#### Retrieve Group Names {#retrieval-gnames}

In case the joining node only knows the group identifier of the group it wishes to join or about which it wishes to get update information from the KDC, the node can contact the KDC to request the corresponding group name and joining resource URI. The node can request several group identifiers at once. It does so by sending a CoAP FETCH request to the /ace-group endpoint at the KDC formatted as defined in {{ace-group-fetch}}.

{{fig-ace-group-fetch}} gives an overview of the exchanges described above, and {{fig-ace-group-fetch-2}} shows an example.


~~~~~~~~~~~
Client                                                     KDC
   |                                                        |
   |-------- Group Name and URI Retrieval Request: -------->|
   |                   FETCH /ace-group                     |
   |                                                        |
   |<-Group Name and URI Retrieval Response: 2.05 (Content)-|
   |                                                        |
~~~~~~~~~~~
{: #fig-ace-group-fetch title="Message Flow of Group Name and URI Retrieval Request-Response" artwork-align="center"}

~~~~~~~~~~~
Request:

Header: FETCH (Code=0.05)
Uri-Host: "kdc.example.com"
Uri-Path: "ace-group"
Content-Format: "application/ace-groupcomm+cbor"
Payload (in CBOR diagnostic notation):
  { "gid": [01, 02] }

Response:

Header: Content (Code=2.05)
Content-Format: "application/ace-groupcomm+cbor"
Payload (in CBOR diagnostic notation):
  { "gid": [01, 02], "gname": ["group1", "group2"],
    "guri": ["ace-group/g1", "ace-group/g2"] }
~~~~~~~~~~~
{: #fig-ace-group-fetch-2 title="Example of Group Name and URI Retrieval Request-Response" artwork-align="center"}

## /ace-group/GROUPNAME

This resource implements the POST and GET and handlers.

### POST Handler {#gid-post}

The POST handler adds the public key of the client to the list of the group members' public keys and returns the symmetric group keying material for the group identified by GROUPNAME. Note that the group joining exchange is done by the client via this operation, as described in {{ssec-key-distribution-exchange}}.

The handler expects a request with payload formatted as a CBOR map, which MAY contain the following fields, which, if included, MUST have the corresponding values:

* 'scope', with value the specific resource at the KDC that the Client is authorized to access, i.e., group or topic name, and role(s). This value is a CBOR byte string wrapping one scope entry, as defined in {{ssec-authorization-request}}.

* 'get_pub_keys', if the Client wishes to receive the public keys of the other nodes in the group from the KDC. This parameter may be present if the KDC stores the public keys of the nodes in the group and distributes them to the Client; it is useless to have here if the set of public keys of the members of the group is known in another way, e.g., it was provided by the AS. Note that including this parameter may result in a large message size for the following response, which can be inconvenient for resource-constrained devices.

  The parameter's value is either the CBOR simple value 'null' (0xf6), or a non-empty CBOR array containing the following three elements.

  - The first element, namely 'inclusion\_flag', encodes the CBOR simple value True.
  
  - The second element, namely 'role\_filter', is a non-empty CBOR array. Each element of the array contains one role or a combination of roles for the group identified by GROUPNAME. The Client indicates that it wishes to receive the public keys of all group members having any of the single roles, or at least all of the roles indicated in any combination of roles. For example, the array \["role1", "role2+role3"\] indicates that the Client wishes to receive the public keys of all group members that have at least "role1" or at least both "role2" and "role3".

  - The third element, namely 'id\_filter', is an empty CBOR array.

  If the Client wishes to receive all public keys of all group members, it specifies the 'get_pub_key' parameter with value the CBOR simple value 'null' (0xf6).

  The CDDL definition {{RFC8610}} of 'get_pub_keys' is given in {{cddl-ex-getpubkeys}}, using as example encoding: node identifier encoded as a CBOR byte string; role identifier encoded as a CBOR text string, and combination of roles encoded as a CBOR array of roles.

  Note that the array of roles 'role\_filter' is non-empty for this handler, but this is not necessarily the case for other handlers using this parameter: if this array is empty, it means that the client is not filtering public keys based on roles.
  
  Also note that the array of node identifiers 'id\_filter' is empty for this handler, because the joining node is not expected or capable to express a filter based on node identifiers at this point in time. Consistently, the 'inclusion\_flag' element is set to the CBOR simple value True. However, the 'id\_filter' array is not necessarily empty for the value of 'get_pub_keys' received by the handler of FETCH to ace-group/GROUPNAME/pub-key (see {{pubkey-fetch}}).

  Finally, the 'get_pub_keys' parameter MUST NOT have the arrays 'role\_filter' and 'id\_filter' as both empty, i.e., in CBOR diagnostic notation: \[ bool, \[ \], \[ \] \]. Thus, if this parameter is received as formatted in that way, it has to be considered malformed.

~~~~~~~~~~~~~~~~~~~~ CDDL
id = bstr

role = tstr

comb_role = [ 2*role ]

inclusion = bool

get_pub_keys = null / [ inclusion, [ *(role / comb_role) ], [ *id ] ]
~~~~~~~~~~~~~~~~~~~~
{: #cddl-ex-getpubkeys title="CDLL definition of get_pub_keys, using as example node identifier encoded as bstr and role as tstr" artwork-align="center"}


* 'client_cred', encoded as a CBOR byte string, with value the original binary representation of the Client's public key. This parameter is used if the KDC is managing (collecting from/distributing to the Client) the public keys of the group members, and if the Client's role in the group will require for it to send messages to one or more group members. It is REQUIRED of the application profiles to define the specific formats that are acceptable to use for encoding public keys in the group (REQ6).

* 'cnonce', encoded as a CBOR byte string, and including a dedicated nonce N_C generated by the Client. This parameter MUST be present if the 'client_cred' parameter is present.

* 'client_cred_verify', encoded as a CBOR byte string. This parameter MUST be present if the 'client_cred' parameter is present and no public key associated to the client's token can be retrieved for that group.

  This parameter contains a proof-of-possession (PoP) evidence computed by the Client over the following PoP input: the scope (encoded as CBOR byte string), concatenated with N_S (encoded as CBOR byte string) concatenated with N_C (encoded as CBOR byte string), where:

    - scope is the CBOR byte string either specified in the 'scope' parameter above, if present, or as a default scope that the handler is expected to understand, if omitted.

    - N_S is the challenge received from the KDC in the 'kdcchallenge' parameter of the 2.01 (Created) response to the Token Transfer Request (see {{token-post}}), encoded as a CBOR byte string.

    - N_C is the nonce generated by the Client and specified in the 'cnonce' parameter above, encoded as a CBOR byte string.

    An example of PoP input to compute 'client_cred_verify' using CBOR encoding is given in {{fig-client-cred-input}}.
  
    A possible type of PoP evidence is a signature, that the Client computes by using its own private key, whose corresponding public key is specified in the 'client_cred' parameter. Application profiles of this specification MUST specify the exact approaches used to compute the PoP evidence to include in 'client_cred_verify', and MUST specify which of those approaches is used in which case (REQ20).

    If the token was not provided to the KDC through a Token Transfer Request (e.g., it is used directly to validate TLS instead), it is REQUIRED of the specific application profile to define how the challenge N_S is generated (REQ21).

* 'pub_keys_repos', which can be present if the format of the Client's public key in the 'client_cred' parameter is a certificate. In such a case, this parameter has as value the URI of the certificate. This parameter is encoded as a CBOR text string. Alternative specific encodings of this parameter MAY be defined in applications of this specification (OPT3).

* 'control_uri', with value a full URI, encoded as a CBOR text string. If 'control_uri' is supported by the Client, the Client acts as a CoAP server and hosts a resource at this specific URI. The KDC MAY use this URI to send CoAP requests to the Client (acting as CoAP server in this exchange), for example for individual provisioning of new keying material when performing a group rekeying (see {{update-keys}}), or to inform the Client of its removal from the group {{sec-node-removal}}. If the KDC does not implement mechanisms using this resource, it can just ignore this parameter. Other additional functionalities of this resource MAY be defined in application profiles of this specifications (OPT9). In particular, this resource is intended for communications concerning exclusively the group or topic specified in the 'scope' parameter.

~~~~~~~~~~~~~~~~~~~~
scope, N_S, and N_C expressed in CBOR diagnostic notation:
      scope = h'826667726F7570316673656E646572'
      N_S = h'018a278f7faab55a'
      N_C = h'25a8991cd700ac01'


scope, N_S, and N_C  as CBOR encoded byte strings:
      scope = 0x4f826667726F7570316673656E646572
      N_S = 0x48018a278f7faab55a
      N_C = 0x4825a8991cd700ac01

PoP input =
  0x4f 826667726F7570316673656E646572
    48 018a278f7faab55a 48 25a8991cd700ac01
~~~~~~~~~~~~~~~~~~~~
{: #fig-client-cred-input title="Example of PoP input to compute 'client_cred_verify' using CBOR encoding" artwork-align="center"}


The handler extracts the granted scope from the access token, and checks the requested one against the token one. If the requested one is not a subset of the token one, the KDC MUST respond with a 4.01 (Unauthorized) error message.

If the request does not include a 'scope' field, the KDC is expected to understand which group and role(s) the Client is requesting (e.g., there is only one the Client has been granted). If the KDC can not recognize which scope the Client is requesting, it MUST respond with a 4.00 (Bad Request) error message.

The KDC verifies that the group name of the /ace-group/GROUPNAME path is a subset of the 'scope' stored in the access token associated to this client. The KDC also verifies that the roles the client is granted in the group allow it to perform this operation on this resource (REQ8). If either verification fails, the KDC MUST respond with a 4.01 (Unauthorized) error message. This response MAY be an AS Request Creation Hints, as defined in {{Section 5.3 of I-D.ietf-ace-oauth-authz}}, in which case the content format MUST be set to application/ace+cbor.

If the request is not formatted correctly (i.e., required fields non received or received with incorrect format), the handler MUST respond with a 4.00 (Bad Request) error message. The response MAY have Content-Format set to application/ace-groupcomm+cbor and have a CBOR map as payload. For instance, the CBOR map can include a 'sign_info' parameter formatted as 'sign_info_res' defined in {{sign-info}}, with the 'pub_key_enc' element set to the CBOR simple value 'null' (0xf6) if the Client sent its own public key and the KDC is not set to store public keys of the group members.

If the request contained unknown or non-expected fields present, the handler MUST silently drop them and continue processing. Application profiles MAY define optional or mandatory payload formats for specific error cases (OPT5).

If the KDC manages the group members' public keys, the handler checks if one is included in the 'client_cred' field. If so, the KDC retrieves the public key and performs the following actions.

* If the access token was provided through a Token Transfer Request (see {{token-post}}) but the KDC cannot retrieve the 'kdcchallenge' associated to this Client (see {{token-post}}), the KDC MUST respond with a 4.00 Bad Request error response, which MUST also have Content-Format application/ace-groupcomm+cbor. The payload of the error response is a CBOR map including a newly generated 'kdcchallenge' value. This is specified in the 'kdcchallenge' parameter, whose CBOR label is defined in {{params}}.

* The KDC checks the public key to be valid for the group identified by GROUPNAME. That is, it checks that the public key is encoded according to the format used in the group, is intended for the public key algorithm used in the group, and is aligned with the possible associated parameters used in the group.

   If this verification fails, the handler MUST respond with a 4.00 (Bad Request) error message. The response MUST have Content-Format set to application/ace-groupcomm+cbor and is formatted as defined in {{key-distr}}. The value of the 'error' field MUST be set to 2 ("Public key incompatible with the group configuration").

* The KDC verifies the PoP evidence contained in the 'client_cred_verify' field. Application profiles of this specification MUST specify the exact approaches used to verify the PoP evidence, and MUST specify which of those approaches is used in which case (REQ20).

   If the PoP evidence does not pass verification, the handler MUST respond with a 4.01 (Unauthorized) error message. The response MUST have Content-Format set to application/ace-groupcomm+cbor and is formatted as defined in {{key-distr}}. The value of the 'error' field MUST be set to 3 ("Invalid Proof-of-Possession evidence").

If no public key is included in the 'client_cred' field, the handler checks if one public key is already associated to the received access token (see {{ssec-key-distribution-exchange}} for an example) and to the group identified by GROUPNAME.

If an eligible public key for the Client is neither present in the 'client_cred' field nor already stored, it is RECOMMENDED that the handler stops the processing and responds with a 4.00 (Bad Request) error message. Applications profiles MAY define alternatives (OPT6).

If all the verifications above succeed, the KDC proceeds as follows.

First, only in case the Client is not already a group member, the handler performs the following actions:

* The handler adds the Client to the list of current members of the group.

* The handler assigns a name NODENAME to the Client, and creates a sub-resource to /ace-group/GROUPNAME at the KDC, i.e., "/ace-group/GROUPNAME/nodes/NODENAME".

* The handler associates the node identifier NODENAME to the access token and the secure session for the Client.

Then, the handler performs the following actions.

* If the KDC manages the group members' public keys:

  - The handler associates the retrieved Client's public key to the node identifier NODENAME and to the access token.

  - The handler adds the retrieved Client's public key to the stored list of public keys stored for the group identified by GROUPNAME. If such list already includes a public key for the Client, but a different public key is specified in the 'client_cred' field, then the handler MUST replace the old public key in the list with the one specified in the 'client_cred' field.

* The handler returns a success response, containing the symmetric group keying material; the group policies; and the public keys of the current members of the group, if the KDC manages those and the Client requested them.

The response message MUST have response code 2.01 (Created) if the Client has been added to the list of group members in this joining exchange (see above), or 2.04 (Changed) otherwise, i.e., if the Client is re-joining the group without having left it.
  
The response message MUST include the Location-Path CoAP option, specifying the URI path to the sub-resource associated to the client, i.e. "/ace-group/GROUPNAME/nodes/NODENAME".

The response message MUST have Content-Format application/ace-groupcomm+cbor. The payload of the response is formatted as a CBOR map, which MUST contain the following fields and values.

* 'gkty', identifying the key type of the 'key' parameter. The set of values can be found in the "Key Type" column of the "ACE Groupcomm Key Types" Registry. Implementations MUST verify that the key type matches the application profile being used, if present, as registered in the "ACE Groupcomm Key Types" registry.

* 'key', containing the keying material for the group communication, or information required to derive it.

* 'num', containing the version number of the keying material for the group communication, formatted as an integer. This is a strictly monotonic increasing field. The application profile MUST define the initial version number (REQ23).

The exact format of the 'key' value MUST be defined in applications of this specification (REQ10), as well as values of 'gkty' accepted by the application (REQ11). Additionally, documents specifying the key format MUST register it in the "ACE Groupcomm Key Types" registry defined in {{iana-key}}, including its name, type and application profile to be used with.

~~~~~~~~~~~
+----------+----------------+---------+-------------------------+
| Name     | Key Type Value | Profile | Description             |
+----------+----------------+---------+-------------------------+
| Reserved | 0              |         | This value is reserved  |
+----------+----------------+---------+-------------------------+
~~~~~~~~~~~
{: #gkty title="Key Type Values" artwork-align="center"}

The response SHOULD contain the following parameter:

* 'exp', with value the expiration time of the keying material for the group communication, encoded as a CBOR unsigned integer. This field contains a numeric value representing the number of seconds from 1970-01-01T00:00:00Z UTC until the specified UTC date/time, ignoring leap seconds, analogous to what specified for NumericDate in {{Section 2 of RFC7519}}. Group members MUST stop using the keying material to protect outgoing messages and retrieve new keying material at the time indicated in this field.

Optionally, the response MAY contain the following parameters, which, if included, MUST have the corresponding values:

* 'ace-groupcomm-profile', with value a CBOR integer that MUST be used to uniquely identify the application profile for group communication. Applications of this specification MUST register an application profile identifier and the related value for this parameter in the "ACE Groupcomm Profiles" Registry (REQ15).

* 'pub\_keys', MUST be present if 'get\_pub\_keys' was present in the request, otherwise it MUST NOT be present. This parameter is a CBOR array specifying the public keys of the group members, i.e., of all of them or of the ones selected according to the 'get\_pub\_keys' parameter in the request. In particular, each element of the array is a CBOR byte string, with value the original binary representation of a group member's public key. It is REQUIRED of the application profiles to define the specific formats of public keys that are acceptable to use in the group (REQ6).

* 'peer\_roles', MUST be present if 'pub\_keys' is also present, otherwise it MUST NOT be present. This parameter is a CBOR array of n elements, with n the number of public keys included in the 'pub\_keys' parameter (at most the number of members in the group). The i-th element of the array specifies the role (or CBOR array of roles) that the group member associated to the i-th public key in 'pub\_keys' has in the group. In particular, each array element is encoded as the role element of a scope entry, as defined in {{ssec-authorization-request}}.

* 'peer\_identifiers', MUST be present if 'pub\_keys' is also present, otherwise it MUST NOT be present. This parameter is a CBOR array of n elements, with n the number of public keys included in the 'pub\_keys' parameter (at most the number of members in the group). The i-th element of the array specifies the node identifier that the group member associated to the i-th public key in 'pub\_keys' has in the group. In particular, the i-th array element is encoded as a CBOR byte string, with value the node identifier of the group member.

* 'group\_policies', with value a CBOR map, whose entries specify how the group handles specific management aspects. These include, for instance, approaches to achieve synchronization of sequence numbers among group members. The elements of this field are registered in the "ACE Groupcomm Policies" Registry. This specification defines the three elements "Sequence Number Synchronization Methods", "Key Update Check Interval" and "Expiration Delta", which are summarized in {{fig-ACE-Groupcomm-Policies}}. Application profiles that build on this document MUST specify the exact content format and default value of included map entries (REQ17).

~~~~~~~~~~~
+--------------+-------+----------|---------------------|------------+
|      Name    | CBOR  |   CBOR   |    Description      | Reference  |
|              | label |   type   |                     |            |
|--------------+-------+----------|---------------------|------------|
| Sequence     | TBD1  | tstr/int | Method for a re-    | [[this     |
| Number       |       |          | cipient node to     | document]] |
| Synchroniza- |       |          | synchronize with    |            |
| tion Method  |       |          | sequence numbers    |            |
|              |       |          | of a sender node.   |            |
|              |       |          | Its value is taken  |            |
|              |       |          | from the 'Value'    |            |
|              |       |          | column of the       |            |
|              |       |          | Sequence Number     |            |
|              |       |          | Synchronization     |            |
|              |       |          | Method registry     |            |
|              |       |          |                     |            |
| Key Update   | TBD2  |   int    | Polling interval    | [[this     |
| Check        |       |          | in seconds, to      | document]] |
| Interval     |       |          | check for new       |            |
|              |       |          | keying material at  |            |
|              |       |          | the KDC             |            |
|              |       |          |                     |            |
| Expiration   | TBD3  |   uint   | Number of seconds   | [[this     |
| Delta        |       |          | from 'exp' until    | document]] |
|              |       |          | the specified UTC   |            |
|              |       |          | date/time after     |            |
|              |       |          | which group members |            |
|              |       |          | MUST stop using the |            |
|              |       |          | keying material to  |            |
|              |       |          | verify incoming     |            |
|              |       |          | messages.           |            |
+--------------+-------+----------|---------------------|------------+
~~~~~~~~~~~
{: #fig-ACE-Groupcomm-Policies title="ACE Groupcomm Policies" artwork-align="center"}

* 'mgt_key_material', encoded as a CBOR byte string and containing the administrative keying material to participate in the group rekeying performed by the KDC. The application profile MUST define if this field is used, and if used then MUST specify the exact format and content which depend on the specific rekeying scheme used in the group. If the usage of ‘mgt_key_material’ is indicated and its format defined for a specific key management scheme, that format must explicitly indicate the key management scheme itself. If a new rekeying scheme is defined to be used for an existing ‘mgt_key_material’ in an existing profile, then that profile will have to be updated accordingly, especially with respect to the usage of ‘mgt_key_material’ related format and content (REQ22).

Specific application profiles that build on this document MUST specify the communication protocol that members of the group use to communicate with each other (REQ13) and how exactly the keying material is used to protect the group communication (REQ14).

CBOR labels for these fields are defined in {{params}}.

#### Join the Group {#ssec-key-distribution-exchange}

{{fig-key-distr-join}} gives an overview of the Joining exchange between Client and KDC, when the Client first joins a group, while {{fig-key-distr-join-2}} shows an example.

~~~~~~~~~~~
Client                                                     KDC
   |                                                        |
   |----- Joining Request: POST /ace-group/GROUPNAME ------>|
   |                                                        |
   |<--------- Joining Response: 2.01 (Created) ----------- |
   | Location-Path = "/ace-group/GROUPNAME/nodes/NODENAME"  |
~~~~~~~~~~~
{: #fig-key-distr-join title="Message Flow of First Exchange for Group Joining" artwork-align="center"}


~~~~~~~~~~~
Request:

Header: POST (Code=0.02)
Uri-Host: "kdc.example.com"
Uri-Path: "ace-group"
Uri-Path: "g1"
Content-Format: "application/ace-groupcomm+cbor"
Payload (in CBOR diagnostic notation,
         with PUB_KEY and POP_EVIDENCE being CBOR byte strings):
  { "scope": << [ "group1", ["sender", "receiver"] ] >> ,
    "get_pub_keys": [true, ["sender"], []], "client_cred": PUB_KEY,
    "cnonce": h'6df49c495409a9b5', "client_cred_verify": POP_EVIDENCE }

Response:

Header: Created (Code=2.01)
Content-Format: "application/ace-groupcomm+cbor"
Location-Path: "kdc.example.com"
Location-Path: "g1"
Location-Path: "nodes"
Location-Path: "c101"
Payload (in CBOR diagnostic notation,
         with KEY being a CBOR byte strings):
  { "gkty": 13, "key": KEY, "num": 12, "exp": 1609459200,
    "pub_keys": [ PUB_KEY1, PUB_KEY2 ],
    "peer_roles": ["sender", ["sender", "receiver"]],
    "peer_identifiers": [ ID1, ID2 ] }
~~~~~~~~~~~
{: #fig-key-distr-join-2 title="Example of First Exchange for Group Joining" artwork-align="center"}


If not previously established, the Client and the KDC MUST first establish a pairwise secure communication channel (REQ19). This can be achieved, for instance, by using a transport profile of ACE. The Joining exchange MUST occur over that secure channel. The Client and the KDC MAY use that same secure channel to protect further pairwise communications that must be secured.

The secure communication protocol is REQUIRED to establish the secure channel between Client and KDC by using the proof-of-possession key bound to the access token. As a result, the proof-of-possession to bind the access token to the Client is performed by using the proof-of-possession key bound to the access token for establishing secure communication between the Client and the KDC.

To join the group, the Client sends a CoAP POST request to the /ace-group/GROUPNAME endpoint at the KDC, where GROUPNAME is the group name of the group to join, formatted as specified in {{gid-post}}. This group name is the same as in the scope entry corresponding to that group, specified in the 'scope' parameter of the Authorization Request/Response, or it can be retrieved from it. Note that, in case of successful joining, the Client will receive the URI to retrieve group keying material and to leave the group in the Location-Path option of the response.

If the node is joining a group for the first time, and the KDC maintains the public keys of the group members, the Client is REQUIRED to send its own public key and proof-of-possession (PoP) evidence ("client_cred" and "client_cred_verify" in {{gid-post}}). The request is accepted only if both public key is provided and the PoP evidence is successfully verified. If a node re-joins a group with the same access token and the same public key, it can omit to send the public key and the PoP evidence, or just omit the PoP evidence, and the KDC will be able to retrieve its public key associated to its token for that group (if the key has been discarded, the KDC will reply with 4.00 Bad Request, as specified in {{gid-post}}). If a node re-joins a group but wants to update its own public key, it needs to send both its public key and the PoP evidence.

If the application requires backward security, the KDC MUST generate new group keying material and securely distribute it to all the current group members, upon a new node's joining the group. To this end, the KDC uses the message format of the response defined in {{gid-get}}. Application profiles may define alternative ways of retrieving the keying material, such as sending separate requests to different resources at the KDC ({{gid-get}}, {{pubkey-get}}, {{policies-get}}). The KDC MUST increment the version number of the current keying material, before distributing the newly generated keying material to the group. After that, the KDC SHOULD store the distributed keying material in persistent storage.

### GET Handler {#gid-get}

The GET handler returns the symmetric group keying material for the group identified by GROUPNAME.

The handler expects a GET request.

The KDC verifies that the group name of the /ace-group/GROUPNAME path is a subset of the 'scope' stored in the access token associated to this client. The KDC also verifies that the roles the client is granted in the group allow it to perform this operation on this resource (REQ8). If either verification fails, the KDC MUST respond with a 4.01 (Unauthorized) error message. This response MAY be an AS Request Creation Hints, as defined in {{Section 5.3 of I-D.ietf-ace-oauth-authz}}, in which case the content format MUST be set to application/ace+cbor.

Additionally, the handler verifies that the node is a current member of the group. If verification fails, the KDC MUST respond with a 4.01 (Unauthorized) error message. The response MUST have Content-Format set to application/ace-groupcomm+cbor and is formatted as defined in {{key-distr}}. The value of the 'error' field MUST be set to 0 ("Operation permitted only to group members").

If verification succeeds, the handler returns a 2.05 (Content) message containing the symmetric group keying material. The payload of the response is formatted as a CBOR map which MUST contain the parameters 'gkty', 'key' and 'num' specified in {{gid-post}}.

The payload MAY also include the parameters 'ace-groupcomm-profile', 'exp', and 'mgt_key_material' parameters specified in {{gid-post}}.

#### Retrieve Group Keying Material {#ssec-key-material-retrieval}

A node in the group can contact the KDC to retrieve the current group keying material, by sending a CoAP GET request to the /ace-group/GROUPNAME endpoint at the KDC, where GROUPNAME is the group name.

{{fig-retrieve-key-material}} gives an overview of the Joining exchange between Client and KDC, when the Client first joins a group, while {{fig-retrieve-key-material-2}} shows an example.

~~~~~~~~~~~
Client                                                              KDC
   |                                                                 |
   |----- Key Distribution Request: POST /ace-group/GROUPNAME ------>|
   |                                                                 |
   |<----------- Key Distribution Response: 2.05 (Content) --------- |
   |                                                                 |
~~~~~~~~~~~
{: #fig-retrieve-key-material title="Message Flow of Key Distribution Request-Response" artwork-align="center"}


~~~~~~~~~~~
Request:

Header: GET (Code=0.01)
Uri-Host: "kdc.example.com"
Uri-Path: "ace-group"
Uri-Path: "g1"
Payload: -

Response:

Header: Content (Code=2.05)
Content-Format: "application/ace-groupcomm+cbor"
Payload (in CBOR diagnostic notation,
         with KEY being a CBOR byte strings):
  { "gkty": 13, "key": KEY, "num": 12 }
~~~~~~~~~~~
{: #fig-retrieve-key-material-2 title="Example of Key Distribution Request-Response" artwork-align="center"}

## /ace-group/GROUPNAME/pub-key

If the KDC does not maintain public keys for the group, the handler for any request on this resource returns a 4.05 (Method Not Allowed) error message. If it does, the rest of this section applies.

This resource implements the GET and FETCH handlers.

### GET Handler {#pubkey-get}

The handler expects a GET request.

The KDC performs the same verifications as the FETCH handler in {{pubkey-fetch}}, and if successful returns the same response as in {{pubkey-fetch}} but without filtering based on roles or node identifiers: all the group members' public keys are returned.

Note that this resource handler, as the FETCH handler for the same resource, only verifies that the node is authorized by the AS to access this resource. Nodes that are not members of the group but are authorized to do signature verifications on the group messages may be allowed to access this resource, if the application needs it.

#### Retrieve All Public Keys in the Group {#sec-key-retrieval-all}

In case the KDC maintains the public keys of group members, a node in the group can contact the KDC to request public keys and roles of all the group members, by sending a CoAP GET request to the /ace-group/GROUPNAME/pub-key endpoint at the KDC, where GROUPNAME is the group name, and formatted as defined in {{pubkey-get}}.

{{fig-public-key-3}} gives an overview of the exchanges mentioned above, while {{fig-public-key-4}} shows an example of such an exchange.


~~~~~~~~~~~
Client                                                     KDC
   |                                                        |
   |--Public Key Request: GET /ace-group/GROUPNAME/pub-key->|
   |                                                        |
   |<--------- Public Key Response: 2.05 (Content) ---------|
   |                                                        |
~~~~~~~~~~~
{: #fig-public-key-3 title="Message Flow of Public Key Exchange to Request the Public Keys of all the Group Members" artwork-align="center"}

~~~~~~~~~~~
Request:

Header: GET (Code=0.01)
Uri-Host: "kdc.example.com"
Uri-Path: "ace-group"
Uri-Path: "g1"
Uri-Path: "pub-key"
Payload: -

Response:

Header: Content (Code=2.05)
Content-Format: "application/ace-groupcomm+cbor"
Payload (in CBOR diagnostic notation):
  { "num": 5,
    "pub_keys": [ PUB_KEY1, PUB_KEY2, PUB_KEY3 ],
    "peer_roles": ["sender", ["sender", "receiver"], "receiver"],
    "peer_identifiers": [ ID1, ID2, ID3 ] }
~~~~~~~~~~~
{: #fig-public-key-4 title="Example of Public Key Exchange to Request the Public Keys of all the Group Members" artwork-align="center"}

### FETCH Handler {#pubkey-fetch}

The FETCH handler receives identifiers of group members for the group identified by GROUPNAME and returns the public keys of such group members.

The handler expects a request with payload formatted as a CBOR map, that MUST contain the following fields:

* 'get_pub_keys', whose value is encoded as in {{gid-post}} with the following modification:

  - The element 'inclusion\_flag' encodes the CBOR simple value True if the third element 'id\_filter' specifies an empty CBOR array, or if the Client wishes to receive the public keys of the nodes having their node identifier specified in 'id\_filter'. Instead, this element encodes the CBOR simple value False if the Client wishes to receive the public keys of the nodes not having the node identifiers specified in the third element 'id\_filter'.

  - The array 'role\_filter' may be empty, if the Client does not wish to filter the requested public keys based on the roles of the group members.

  - The array 'id\_filter' contains zero or more node identifiers of group members, for the group identified by GROUPNAME. The Client indicates that it wishes to receive the public keys of the nodes having or not having these node identifiers, in case the 'inclusion\_flag' parameter encodes the CBOR simple value True or False, respectively. The array may be empty, if the Client does not wish to filter the requested public keys based on the node identifiers of the group members.

Note that, in case both the 'role\_filter' array and the 'id\_filter' array are non-empty:

* If the 'inclusion\_flag' encodes the CBOR simple value True, the handler returns the public keys of group members whose roles match with 'role\_filter' and/or having their node identifier specified in 'id\_filter'.

* If the 'inclusion\_flag' encodes the CBOR simple value False, the handler returns the public keys of group members whose roles match with 'role\_filter' and, at the same time, not having their node identifier specified in 'id\_filter'.

Finally, as mentioned in {{gid-post}}, the arrays 'role\_filter' and 'id\_filter' MUST NOT both be empty.

The specific format of public keys as well as identifiers, roles and combination of roles of group members MUST be specified by the application profile (OPT1, REQ2, REQ12).

The KDC verifies that the group name of the /ace-group/GROUPNAME path is a subset of the 'scope' stored in the access token associated to this client. The KDC also verifies that the roles the client is granted in the group allow it to perform this operation on this resource (REQ8). If either verification fails, the KDC MUST respond with a 4.01 (Unauthorized) error message.

If verification succeeds, the handler identifies the public keys of the current group members for which either:

  - the role identifier matches with one of those indicated in the request; note that the request can contain a "combination of roles", where the handler select all group members who have all roles included in the combination.

  - the node identifier matches with one of those indicated in the request.

Then, the handler returns a 2.05 (Content) message response with payload formatted as a CBOR map, containing only the following parameters from {{gid-post}}.

* 'num', which encodes the version number of the current group keying material.

* 'pub\_keys', which encodes the list of public keys of the selected group members.

* 'peer\_roles', which encodes the role (or CBOR array of roles) that each of the selected group members has in the group.

* 'peer\_identifiers', which encodes the node identifier that each of the selected group members has in the group.

The specific format of public keys as well as of node identifiers of group members is specified by the application profile (REQ6, REQ12).

If the KDC does not store any public key associated to the specified node identifiers, the handler returns a response with payload formatted as a CBOR byte string of zero length.

The handler MAY enforce one of the following policies, in order to handle possible node identifiers that are included in the 'id_filter' element of the 'get_pub_keys' parameter of the request but are not associated to any current group member. Such a policy MUST be specified by the application profile (REQ16).

* The KDC silently ignores those node identifiers.

* The KDC retains public keys of group members for a given amount of time after their leaving, before discarding them. As long as such public keys are retained, the KDC provides them to a requesting Client.

Note that this resource handler only verifies that the node is authorized by the AS to access this resource. Nodes that are not members of the group but are authorized to do signature verifications on the group messages may be allowed to access this resource, if the application needs it.

#### Retrieve a Subset of Public Keys in the Group {#sec-key-retrieval}

In case the KDC maintains the public keys of group members, a node in the group can contact the KDC to request public keys and roles of a specified subset of group members, by sending a CoAP FETCH request to the /ace-group/GROUPNAME/pub-key endpoint at the KDC, where GROUPNAME is the group name, and formatted as defined in {{pubkey-fetch}}.

{{fig-public-key-1}} gives an overview of the exchange mentioned above, while {{fig-public-key-2}} shows an example of such an exchange.

~~~~~~~~~~~
Client                                                      KDC
   |                                                         |
   |-Public Key Request: FETCH /ace-group/GROUPNAME/pub-key->|
   |                                                         |
   |<--------- Public Key Response: 2.05 (Created) ----------|
   |                                                         |
~~~~~~~~~~~
{: #fig-public-key-1 title="Message Flow of Public Key Exchange to Request the Public Keys of Specific Group Members" artwork-align="center"}


~~~~~~~~~~~
Request:

Header: FETCH (Code=0.05)
Uri-Host: "kdc.example.com"
Uri-Path: "ace-group"
Uri-Path: "g1"
Uri-Path: "pub-key"
Content-Format: "application/ace-groupcomm+cbor"
Payload:
  { "get_pub_keys": [true, [], [ ID3 ]] }

Response:

Header: Content (Code=2.05)
Content-Format: "application/ace-groupcomm+cbor"
Payload (in CBOR diagnostic notation):
  { "pub_keys": [ PUB_KEY3 ],
    "peer_roles": [ "receiver" ],
    "peer_identifiers": [ ID3 ] }
~~~~~~~~~~~
{: #fig-public-key-2 title="Example of Public Key Exchange to Request the Public Keys of Specific Group Members" artwork-align="center"}

## /ace-group/GROUPNAME/policies

This resource implements the GET handler.

### GET Handler {#policies-get}

The handler expects a GET request.

The KDC verifies that the group name of the /ace-group/GROUPNAME path is a subset of the 'scope' stored in the access token associated to this client. The KDC also verifies that the roles the client is granted in the group allow it to perform this operation on this resource (REQ8). If either verification fails, the KDC MUST respond with a 4.01 (Unauthorized) error message.

Additionally, the handler verifies that the node is a current member of the group. If verification fails, the KDC MUST respond with a 4.01 (Unauthorized) error message. The response MUST have Content-Format set to application/ace-groupcomm+cbor and is formatted as defined in {{key-distr}}. The value of the 'error' field MUST be set to 0 ("Operation permitted only to group members").

If verification succeeds, the handler returns a 2.05 (Content) message containing the list of policies for the group identified by GROUPNAME. The payload of the response is formatted as a CBOR map including only the parameter 'group_policies' defined in {{gid-post}} and specifying the current policies in the group. If the KDC does not store any policy, the payload is formatted as a zero-length CBOR byte string.

The specific format and meaning of group policies MUST be specified in the application profile (REQ17).

#### Retrieve the Group Policies {#policies}

A node in the group can contact the KDC to retrieve the current group policies, by sending a CoAP GET request to the /ace-group/GROUPNAME/policies endpoint at the KDC, where GROUPNAME is the group name, and formatted as defined in {{policies-get}}

{{fig-policies}} gives an overview of the exchange described above, while {{fig-policies-2}} shows an example.

~~~~~~~~~~~
Client                                                   KDC
   |                                                      |
   |-Policies Request: GET ace-group/GROUPNAME/policies ->|
   |                                                      |
   |<--------- Policies Response: 2.05 (Content) ---------|
   |                                                      |
~~~~~~~~~~~
{: #fig-policies title="Message Flow of Policies Request-Response" artwork-align="center"}


~~~~~~~~~~~
Request:

Header: GET (Code=0.01)
Uri-Host: "kdc.example.com"
Uri-Path: "ace-group"
Uri-Path: "g1"
Uri-Path: "policies"
Payload: -

Response:

Header: Content (Code=2.05)
Content-Format: "application/ace-groupcomm+cbor"
Payload(in CBOR diagnostic notation):
  { "group_policies": {"exp-delta": 120} }
~~~~~~~~~~~
{: #fig-policies-2 title="Example of Policies Request-Response" artwork-align="center"}

## /ace-group/GROUPNAME/num

This resource implements the GET handler.

### GET Handler {#num-get}

The handler expects a GET request.

The KDC verifies that the group name of the /ace-group/GROUPNAME path is a subset of the 'scope' stored in the access token associated to this client. The KDC also verifies that the roles the client is granted in the group allow it to perform this operation on this resource (REQ8). If either verification fails, the KDC MUST respond with a 4.01 (Unauthorized) error message.

Additionally, the handler verifies that the node is a current member of the group. If verification fails, the KDC MUST respond with a 4.01 (Unauthorized) error message. The response MUST have Content-Format set to application/ace-groupcomm+cbor and is formatted as defined in {{key-distr}}. The value of the 'error' field MUST be set to 0 ("Operation permitted only to group members").

If verification succeeds, the handler returns a 2.05 (Content) message containing an integer that represents the version number of the symmetric group keying material. This number is incremented on the KDC every time the KDC updates the symmetric group keying material, before the new keying material is distributed. This number is stored in persistent storage.

The payload of the response is formatted as a CBOR integer.

#### Retrieve the Keying Material Version {#key-version}

A node in the group can contact the KDC to request information about the version number of the symmetric group keying material, by sending a CoAP GET request to the /ace-group/GROUPNAME/num endpoint at the KDC, where GROUPNAME is the group name, formatted as defined in {{num-get}}. In particular, the version is incremented by the KDC every time the group keying material is renewed, before it's distributed to the group members.

{{fig-version}} gives an overview of the exchange described above, while {{fig-version-2}} shows an example.

~~~~~~~~~~~
Client                                                    KDC
   |                                                       |
   |---- Version Request: GET ace-group/GROUPNAME/num ---->|
   |                                                       |
   |<--------- Version Response: 2.05 (Content) -----------|
   |                                                       |
~~~~~~~~~~~
{: #fig-version title="Message Flow of Version Request-Response" artwork-align="center"}


~~~~~~~~~~~
Request:

Header: GET (Code=0.01)
Uri-Host: "kdc.example.com"
Uri-Path: "ace-group"
Uri-Path: "g1"
Uri-Path: "num"
Payload: -

Response:

Header: Content (Code=2.05)
Content-Format: "application/ace-groupcomm+cbor"
Payload(in CBOR diagnostic notation):
  13
~~~~~~~~~~~
{: #fig-version-2 title="Example of Version Request-Response" artwork-align="center"}

## /ace-group/GROUPNAME/nodes/NODENAME

This resource implements the GET, PUT and DELETE handlers.

### GET Handler {#node-get}

The handler expects a GET request.

The KDC verifies that the group name of the /ace-group/GROUPNAME path is a subset of the 'scope' stored in the access token associated to the client identified by NODENAME. The KDC also verifies that the roles the client is granted in the group allow it to perform this operation on this resource (REQ8). If either verification fails, the KDC MUST respond with a 4.01 (Unauthorized) error message.

The handler also verifies that the node sending the request and the node name used in the Uri-Path match. If that is not the case, the handler responds with a 4.01 (Unauthorized) error response.

Additionally, the handler verifies that the node is a current member of the group. If verification fails, the KDC MUST respond with a 4.01 (Unauthorized) error message. The response MUST have Content-Format set to application/ace-groupcomm+cbor and is formatted as defined in {{key-distr}}. The value of the 'error' field MUST be set to 0 ("Operation permitted only to group members").

If verification succeeds, the handler returns a 2.05 (Content) message containing both the group keying material and the individual keying material for the Client, or information enabling the Client to derive it. The payload of the response is formatted as a CBOR map. The format for the group keying material is the same as defined in the response of {{gid-get}}. The specific format of individual keying material for group members, or of the information to derive it, and corresponding CBOR label, MUST be specified in the application profile (REQ18) and registered in {{iana-reg}}.

Optionally, the KDC can make the sub-resource at ace-group/GROUPNAME/nodes/NODENAME also Observable {{RFC7641}} for the associated node. In case the KDC removes that node from the group without having been explicitly asked for it, this allows the KDC to send an unsolicited 4.04 (Not Found) response to the node as a notification of eviction from the group (see {{sec-node-removal}}).

Note that the node could have been observing also the resource at ace-group/GROUPNAME, in order to be informed of changes in the keying material. In such a case, this method would result in largely overlapping notifications received for the resource at ace-group/GROUPNAME and the sub-resource at ace-group/GROUPNAME/nodes/NODENAME.
   
In order to mitigate this, a node that supports the No-Response option {{RFC7967}} can use it when starting the observation of the sub-resource at ace-group/GROUPNAME/nodes/NODENAME. In particular, the GET observation request can also include the No-Response option, with value set to 2 (Not interested in 2.xx responses).

#### Retrieve Group and Individual Keying Material {#update-keys}

When any of the following happens, a node MUST stop using the owned group keying material to protect outgoing messages, and SHOULD stop using it to decrypt and verify incoming messages.

* Upon expiration of the keying material, according to what indicated by the KDC with the 'exp' parameter in a Joining Response, or to a pre-configured value.

* Upon receiving a notification of revoked/renewed keying material from the KDC, possibly as part of an update of the keying material (rekeying) triggered by the KDC.

* Upon receiving messages from other group members without being able to retrieve the keying material to correctly decrypt them. This may be due to rekeying messages previously sent by the KDC, that the Client was not able to receive or decrypt.

In either case, if it wants to continue participating in the group communication, the node has to request the latest keying material from the KDC. To this end, the Client sends a CoAP GET request to the /ace-group/GROUPNAME/nodes/NODENAME endpoint at the KDC, formatted as specified in {{node-get}}.

Note that policies can be set up, so that the Client sends a Key Re-Distribution request to the KDC only after a given number of received messages could not be decrypted (because of failed decryption processing or inability to retrieve the necessary keying material).

It is application dependent and pertaining to the particular message exchange (e.g., {{I-D.ietf-core-oscore-groupcomm}}) to set up these policies for instructing clients to retain incoming messages and for how long (OPT4). This allows clients to possibly decrypt such messages after getting updated keying material, rather than just consider them non valid messages to discard right away.

The same Key Distribution Request could also be sent by the Client without being triggered by a failed decryption of a message, if the Client wants to be sure that it has the latest group keying material. If that is the case, the Client will receive from the KDC the same group keying material it already has in memory.

{{fig-key-redistr-req-resp}} gives an overview of the exchange described above, while {{fig-key-redistr-req-resp-2}} shows an example.

~~~~~~~~~~~
Client                                                          KDC
   |                                                             |
   |------------------ Key Distribution Request: --------------->|
   |           GET ace-group/GROUPNAME/nodes/NODENAME            |
   |                                                             |
   |<-------- Key Distribution Response: 2.05 (Content) ---------|
   |                                                             |
~~~~~~~~~~~
{: #fig-key-redistr-req-resp title="Message Flow of Key Distribution Request-Response" artwork-align="center"}


~~~~~~~~~~~
Request:

Header: GET (Code=0.01)
Uri-Host: "kdc.example.com"
Uri-Path: "ace-group"
Uri-Path: "g1"
Uri-Path: "nodes"
Uri-Path: "c101"
Payload: -

Response:

Header: Content (Code=2.05)
Content-Format: "application/ace-groupcomm+cbor"
Payload (in CBOR diagnostic notation,
         with KEY and IND_KEY being CBOR byte strings,
         and "ind-key" the profile-specified label
         for individual keying material):
  { "gkty": 13, "key": KEY, "num": 12, "ind-key": IND_KEY }
~~~~~~~~~~~
{: #fig-key-redistr-req-resp-2 title="Example of Key Distribution Request-Response" artwork-align="center"}

Alternatively, the re-distribution of keying material can be initiated by the KDC, which e.g.,:

* Can make the ace-group/GROUPNAME resource Observable {{RFC7641}}, and send notifications to observer Clients when the keying material is updated.

   In case the KDC deletes the group identified by GROUPNAME, this also allows the KDC to send an unsolicited 4.04 (Not Found) response to each observer group member, as a notification of group termination. The response MUST have Content-Format set to application/ace-groupcomm+cbor and is formatted as defined in {{key-distr}}. The value of the 'error' field MUST be set to 6 ("Group deleted").

* Can send the payload of the Key Distribution Response in one or multiple multicast POST requests to the members of the group, using secure rekeying schemes such as {{RFC2093}}{{RFC2094}}{{RFC2627}}.

* Can send unicast POST requests to each Client over a secure channel, with the same payload as the Key Distribution Response. When sending such requests, the KDC can target the URI path provided by the intended recipient upon joining the group, as specified in the 'control_uri' parameter of the Joining Request (see {{gid-post}}).

* Can act as a publisher in a pub-sub scenario, and update the keying material by publishing on a specific topic on a broker, which all the members of the group are subscribed to.

Note that these methods of KDC-initiated key distribution have different security properties and require different security associations.

### PUT Handler {#node-put}

The PUT handler is used to get the KDC to produce and return individual keying material to protect outgoing messages for the node (identified by NODENAME) for the group identified by GROUPNAME. Application profiles MAY also use this handler to rekey the whole group. It is up to the application profiles to specify if this handler supports renewal of individual keying material, renewal of the group keying material or both (OPT8).

The handler expects a request with empty payload. In case the request has a non-empty payload, the KDC MUST respond with a 4.00 (Bad Request) error message.

The KDC verifies that the group name of the /ace-group/GROUPNAME path is a subset of the 'scope' stored in the access token associated to the client identified by NODENAME. The KDC also verifies that the roles the client is granted in the group allow it to perform this operation on this resource (REQ8). If either verification fails, the KDC MUST respond with a 4.01 (Unauthorized) error message.

The handler also verifies that the node sending the request and the node name used in the Uri-Path match. If that is not the case, the handler responds with a 4.01 (Unauthorized) error response.

Additionally, the handler verifies that the node is a current member of the group. If the verification fails, the KDC MUST respond with a 4.01 (Unauthorized) error message. The response MUST have Content-Format set to application/ace-groupcomm+cbor and is formatted as defined in {{key-distr}}. The value of the 'error' field MUST be set to 0 ("Operation permitted only to group members").

Also, the handler verifies that this operation is consistent with the set of roles that the node has in the group. If the verification fails, the KDC MUST respond with a 4.00 (Bad Request) error message. The response MUST have Content-Format set to application/ace-groupcomm+cbor and is formatted as defined in {{key-distr}}. The value of the 'error' field MUST be set to 1 ("Request inconsistent with the current roles").

If the KDC is currently not able to serve this request, i.e., to generate new individual keying material for the requesting client, the KDC MUST respond with a 5.03 (Service Unavailable) error message. The response MUST have Content-Format set to application/ace-groupcomm+cbor and is formatted as defined in {{key-distr}}. The value of the 'error' field MUST be set to 4 ("No available node identifiers").

If all verifications succeed, the handler returns a 2.05 (Content) message containing newly-generated keying material for the Client, and/or, if the application profiles requires it (OPT8), starts the complete group rekeying.
The payload of the response is formatted as a CBOR map. The specific format of newly-generated individual keying material for group members, or of the information to derive it, and corresponding CBOR label, MUST be specified in the application profile (REQ18) and registered in {{iana-reg}}.

#### Request to Change Individual Keying Material {#new-keys}

Beside possible expiration, the client may need to communicate to the KDC its need for the keying material to be renewed, e.g., due to exhaustion of AEAD nonces, if AEAD is used for protecting group communication. Depending on the application profile (OPT8), this can result in renewal of individual keying material, group keying material, or both.

For example, if the Client uses an individual key to protect outgoing traffic and has to renew it, the node may request a new one, or new input material to derive it, without renewing the whole group keying material.

To this end, the client performs a Key Renewal Request/Response exchange with the KDC, i.e., it sends a CoAP PUT request to the /ace-group/GROUPNAME/nodes/NODENAME endpoint at the KDC, where GROUPNAME is the group name and NODENAME is its node name, and formatted as defined in {{node-get}}.

{{fig-renewal-req-resp}} gives an overview of the exchange described above, while {{fig-renewal-req-resp-2}} shows an example.

~~~~~~~~~~~
Client                                                    KDC
   |                                                       |
   |------------------ Key Renewal Request: -------------->|
   |           PUT ace-group/GROUPNAME/nodes/NODENAME      |
   |                                                       |
   |<-------- Key Renewal Response: 2.05 (Content) --------|
   |                                                       |
~~~~~~~~~~~
{: #fig-renewal-req-resp title="Message Flow of Key Renewal Request-Response" artwork-align="center"}

~~~~~~~~~~~
Request:

Header: PUT (Code=0.03)
Uri-Host: "kdc.example.com"
Uri-Path: "ace-group"
Uri-Path: "g1"
Uri-Path: "nodes"
Uri-Path: "c101"
Payload: -

Response:

Header: Content (Code=2.05)
Content-Format: "application/ace-groupcomm+cbor"
Payload (in CBOR diagnostic notation, with IND_KEY being
         a CBOR byte string, and "ind-key" the profile-specified
         label for individual keying material):
  { "ind-key": IND_KEY }
~~~~~~~~~~~
{: #fig-renewal-req-resp-2 title="Example of Key Renewal Request-Response" artwork-align="center"}

Note the difference between the Key Distribution Request and the Key Renewal Request: while the first one only triggers distribution (the renewal might have happened independently, e.g., because of expiration), the second one triggers the KDC to produce new individual keying material for the requesting node.

### DELETE Handler {#node-delete}

The DELETE handler removes the node identified by NODENAME from the group identified by GROUPNAME.

The handler expects a request with method DELETE (and empty payload).

The handler verifies that the group name of the /ace-group/GROUPNAME
path is a subset of the 'scope' stored in the access token associated
to the client identified by NODENAME. If the verification fails, the
KDC MUST respond with a 4.01 (Unauthorized) error message.

The handler also verifies that the node sending the request and the node name used in the Uri-Path match. If that is not the case, the handler responds with a 4.01 (Unauthorized) error response.

Additionally, the handler verifies that the node is a current member of the group. If verification fails, the KDC MUST respond with a 4.01 (Unauthorized) error message. The response MUST have Content-Format set to application/ace-groupcomm+cbor and is formatted as defined in {{key-distr}}. The value of the 'error' field MUST be set to 0 ("Operation permitted only to group members").

If verification succeeds, the handler removes the client from the group identified by GROUPNAME. That includes removing the public key of the client if the KDC keep tracks of that, and possibly removing the evicted node from the list of observers of the resource at ace-group/GROUPNAME (if observable). Then, the handler deletes the sub-resource nodes/NODENAME and returns a 2.02 (Deleted) message with empty payload.

#### Leave the Group ## {#ssec-group-leaving}

A node can actively request to leave the group. In this case, the Client sends a CoAP DELETE request to the endpoint /ace-group/GROUPNAME/nodes/NODENAME at the KDC, where GROUPNAME is the group name and NODENAME is its node name, formatted as defined in {{node-delete}}

Alternatively, a node may be removed by the KDC, without having explicitly asked for it. This is further discussed in {{sec-node-removal}}.

## /ace-group/GROUPNAME/nodes/NODENAME/pub-key

This resource implements the POST handler.

### POST Handler {#node-pub-key-post}

The POST handler is used to replace the stored public key of this client (identified by NODENAME) with the one specified in the request at the KDC, for the group identified by GROUPNAME.

The handler expects a POST request with payload as specified in {{gid-post}}, with the difference that it includes only the parameters 'client_cred', 'cnonce' and 'client_cred_verify'. In particular, the PoP evidence included in 'client_cred_verify' is computed in the same way considered in {{gid-post}} and defined by the specific application profile (REQ20), with a newly generated N_C nonce and the previously received N_S. It is REQUIRED of the application profiles to define the specific formats of public keys that are acceptable to use in the group (REQ6).

The handler verifies that the group name GROUPNAME is a subset of the 'scope' stored in the access token associated to the client identified by NODENAME. The KDC also verifies that the roles the client is granted in the group allow it to perform this operation on this resource (REQ8). If either verification fails, the KDC MUST respond with a 4.01 (Unauthorized) error message.

The handler also verifies that the node sending the request and the node name used in the Uri-Path match. If that is not the case, the handler responds with a 4.01 (Unauthorized) error response.

Additionally, the handler verifies that the node is a current member of the group. If the verification fails, the KDC MUST respond with a 4.01 (Unauthorized) error message. The response MUST have Content-Format set to application/ace-groupcomm+cbor and is formatted as defined in {{key-distr}}. The value of the 'error' field MUST be set to 0 ("Operation permitted only to group members").

Also, the handler verifies that this operation is consistent with the set of roles that the node has in the group. If the verification fails, the KDC MUST respond with a 4.00 (Bad Request) error message. The response MUST have Content-Format set to application/ace-groupcomm+cbor and is formatted as defined in {{key-distr}}. The value of the 'error' field MUST be set to 1 ("Request inconsistent with the current roles")

If the request is not formatted correctly (i.e., required fields non received or received with incorrect format), the handler MUST respond with a 4.00 (Bad Request) error message. If the request contains unknown or non-expected fields present, the handler MUST silently ignore them and continue processing. Application profiles MAY define optional or mandatory payload formats for specific error cases (OPT5).

If the KDC cannot retrieve the 'kdcchallenge' associated to this Client (see {{token-post}}), the KDC MUST respond with a 4.00 Bad Request error response, which MUST also have Content-Format application/ace-groupcomm+cbor. The payload of the error response is a CBOR map including a newly generated 'kdcchallenge' value. This is specified in the 'kdcchallenge' parameter, whose CBOR label is defined in {{params}}. In such a case the KDC MUST store the newly generated value as the 'kdcchallenge' value associated to this Client, possibly replacing the currently stored value.

Otherwise, the handler checks that the public key specified in the 'client_cred' field is valid for the group identified by GROUPNAME. That is, the handler checks that the public key is encoded according to the format used in the group, is intended for the public key algorithm used in the group, and is aligned with the possible associated parameters used in the group. If that cannot be successfully verified, the handler MUST respond with a 4.00 (Bad Request) error message. The response MUST have Content-Format set to application/ace-groupcomm+cbor and is formatted as defined in {{key-distr}}. The value of the 'error' field MUST be set to 2 ("Public key incompatible with the group configuration").

Otherwise, the handler verifies the PoP evidence contained in the 'client_cred_verify' field of the request, by using the public key specified in the 'client_cred' field, as well as the same way considered in {{gid-post}} and defined by the specific application profile (REQ20). If the PoP evidence does not pass verification, the handler MUST respond with a 4.01 (Unauthorized) error message. The response MUST have Content-Format set to application/ace-groupcomm+cbor and is formatted as defined in {{key-distr}}. The value of the 'error' field MUST be set to 3 ("Invalid Proof-of-Possession evidence").

If verification succeeds, the handler performs the following actions.

* The handler associates the public key from the 'client_cred' field of the request to the node identifier NODENAME and to the access token associated to the node identified by NODENAME.

* In the stored list of group members' public keys for the group identified by GROUPNAME, the handler replaces the public key of the node identified by NODENAME with the public key specified in the 'client_cred' field of the request.

Then, the handler replies with a 2.04 (Changed) response, which does not include a payload.

#### Uploading a New Public Key {#update-pub-key}

In case the KDC maintains the public keys of group members, a node in the group can contact the KDC to upload a new public key to use in the group, and replace the currently stored one.

To this end, the Client performs a Public Key Update Request/Response exchange with the KDC, i.e., it sends a CoAP POST request to the /ace-group/GROUPNAME/nodes/NODENAME/pub-key endpoint at the KDC, where GROUPNAME is the group name and NODENAME is its node name.

The request is formatted as specified in {{node-pub-key-post}}.

Figure {{fig-pub-key-update-req-resp}} gives an overview of the exchange described above, while {{fig-pub-key-update-req-resp-2}} shows an example.

~~~~~~~~~~~
Client                                                           KDC
|                                                                 |
|-------------- Public Key Update Request: ---------------------->|
|      POST ace-group/GROUPNAME/nodes/NODENAME/pub-key            |
|                                                                 |
|<------- Public Key Update Response: 2.04 (Changed) -------------|
|                                                                 |
~~~~~~~~~~~
{: #fig-pub-key-update-req-resp title="Message Flow of Public Key Update Request-Response" artwork-align="center"}


~~~~~~~~~~~
Request:

Header: POST (Code=0.02)
Uri-Host: "kdc.example.com"
Uri-Path: "ace-group"
Uri-Path: "g1"
Uri-Path: "nodes"
Uri-Path: "c101"
Uri-Path: "pub-key"
Content-Format: "application/ace-groupcomm+cbor"
Payload (in CBOR diagnostic notation, with PUB_KEY
         and POP_EVIDENCE being CBOR byte strings):
  { "client_cred": PUB_KEY, "cnonce": h'9ff7684414affcc8',
    "client_cred_verify": POP_EVIDENCE }

Response:

Header: Changed (Code=2.04)
Payload: -
~~~~~~~~~~~
{: #fig-pub-key-update-req-resp-2 title="Example of Public Key Update Request-Response" artwork-align="center"}

If the application requires backward security, the KDC MUST generate new group keying material and securely distribute it to all the current group members, upon a group member updating its own public key. To this end, the KDC uses the message format of the response defined in {{gid-get}}. Application profiles may define alternative ways of retrieving the keying material, such as sending separate requests to different resources at the KDC ({{gid-get}}, {{pubkey-get}}, {{policies-get}}).
The KDC MUST increment the version number of the current keying material, before distributing the newly generated keying material to the group. After that, the KDC SHOULD store the distributed keying material in persistent storage.

Additionally, after updating its own public key, a group member MAY send a number of the later requests including an identifier of the updated public key, to signal nodes that they need to retrieve it. How that is done depends on the group communication protocol used, and therefore is application profile specific (OPT10).

# Removal of a Node from the Group {#sec-node-removal}

This section describes the different scenarios according to which a node ends up being removed from the group.

If the application requires forward security, the KDC MUST generate new group keying material and securely distribute it to all the current group members but the leaving node, using the message format of the Key Distribution Response (see {{update-keys}}). Application profiles may define alternative message formats. The KDC MUST increment the version number of the current keying material, before distributing the newly generated keying material to the group. After that, the KDC SHOULD store the distributed keying material in persistent storage.

Note that, after having left the group, a node may wish to join it again. Then, as long as the node is still authorized to join the group, i.e., it still has a valid access token, it can request to re-join the group directly to the KDC without needing to retrieve a new access token from the AS. This means that the KDC might decide to keep track of nodes with valid access tokens, before deleting all information about the leaving node.

A node may be evicted from the group in the following cases.

1. The node explicitly asks to leave the group, as defined in {{ssec-group-leaving}}.

2. The node has been found compromised or is suspected so.

3. The node's authorization to be a group member is not valid anymore, either because the access token has expired, or it has been revoked. If the AS provides Token introspection (see {{Section 5.9 of I-D.ietf-ace-oauth-authz}}), the KDC can optionally use it and check whether the node is still authorized for that group in that role.

   In either case, once aware that a node is not authorized anymore, the KDC has to remove the unauthorized node from the list of group members, if the KDC keeps track of that.

Furthermore, in case of forced eviction, the KDC removes the public key of the evicted node if the KDC keep tracks of that, and possibly removes the evicted node from the list of observers of the resource at ace-group/GROUPNAME (if observable).

Then, the KDC deletes the sub-resource ace-group/GROUPNAME/nodes/NODENAME associated to the evicted node. After that, the KDC MAY explicitly inform the evicted node, by means of the following methods.

* If the evicted node implements the 'control_uri' resource specified in {{gid-post}}, the KDC sends a DELETE request, targeting the URI specified in the 'control_uri' parameter of the Joining Request (see {{gid-post}}).
   
* If the evicted node is observing its associated sub-resource at ace-group/GROUPNAME/nodes/NODENAME (see {{node-get}}), the KDC sends an unsolicited 4.04 (Not Found) response, which does not include the Observe option and indicates that the observed resource has been deleted (see {{Section 3.2 of RFC7641}}).

   The response MUST have Content-Format set to application/ace-groupcomm+cbor and is formatted as defined in {{key-distr}}. The value of the 'error' field MUST be set to 5 ("Group membership terminated").

   Consistently, the KDC also removes the node's entry from the list of observers of the sub-resource.

# Extended Scope Format # {#sec-extended-scope}

This section defines an extended format of binary encoded scope, which additionally specifies the semantics used to express the same access control information from the corresponding original scope.

As also discussed in {{ssec-authorization-response}}, this enables a Resource Server to unambiguously process a received access token, also in case the Resource Server runs multiple applications or application profiles that involve different scope semantics.

The extended format is intended only for the 'scope' claim of access tokens, for the cases where the claim takes as value a CBOR byte string. That is, the extended format does not apply to the 'scope' parameter included in ACE messages, i.e., the Authorization Request and Authorization Response exchanged between the client and the Authorization Server (see {{Sections 5.8.1 and 5.8.2 of I-D.ietf-ace-oauth-authz}}), the AS Request Creation Hints message from the Resource Server (see {{Section 5.3 of I-D.ietf-ace-oauth-authz}}), and the Introspection Response from the Authorization Server (see {{Section 5.9.2 of I-D.ietf-ace-oauth-authz}}).

The value of the 'scope' claim following the extended format is composed as follows. Given the original scope using a semantics SEM and encoded as a CBOR byte string, the corresponding extended scope is encoded as a tagged CBOR byte string, wrapping a CBOR sequence {{RFC8742}} of two elements. In particular:

* The first element of the sequence is a CBOR integer, and identifies the semantics SEM used for this scope. The value of this element has to be taken from the "Value" column of the "ACE Scope Semantics" registry defined in {{iana-scope-semantics}} of this specification.

   When defining a new semantics for a binary scope, it is up to the applications and application profiles to define and register the corresponding integer identifier (REQ24).

* The second element of the sequence is the original scope using the semantics SEM, encoded as a CBOR byte string.

Finally, the CBOR byte string wrapping the CBOR sequence is tagged, and identified by the CBOR tag TBD_TAG "ACE Extended Scope Format", defined in {{iana-cbor-tags}} of this specification.

The resulting tagged CBOR byte string is used as value of the 'scope' claim of the access token.

The usage of the extended scope format is not limited to application profiles of this specification or to applications based on group communication. Rather, it is generally applicable to any application and application profile where access control information in the access token is expressed as a binary encoded scope.

{{cddl-ex-0-ext}} and {{cddl-ex-ext}} build on the examples in {{ssec-authorization-response}}, and show the corresponding extended scopes.

~~~~~~~~~~~~~~~~~~~~ CDDL
gname = tstr

permissions = uint . bits roles

roles = &(
   Requester: 1,
   Responder: 2,
   Monitor: 3,
   Verifier: 4
)

scope_entry = AIF_Generic<gname, permissions>

scope = << [ + scope_entry ] >>

semantics = int

; This defines an array, the elements
; of which are to be used in a CBOR Sequence:
sequence = [semantics, scope]

extended_scope = #6.TBD_TAG(<< sequence >>)
~~~~~~~~~~~~~~~~~~~~
{: #cddl-ex-0-ext title="Example CDLL definition of scope, using the default Authorization Information Format"}

~~~~~~~~~~~~~~~~~~~~ CDDL
gname = tstr

role = tstr

scope_entry = [ gname , ? ( role / [ 2*role ] ) ]

scope = << [ + scope_entry ] >>

semantics = int

; This defines an array, the elements
; of which are to be used in a CBOR Sequence:
sequence = [semantics, scope]

extended_scope = #6.TBD_TAG(<< sequence >>)
~~~~~~~~~~~~~~~~~~~~
{: #cddl-ex-ext title="CDLL definition of scope, using as example group name encoded as tstr and role as tstr"}

# ACE Groupcomm Parameters {#params}

This specification defines a number of fields used during the second part of the message exchange, after the exchange of Token Transfer Request and Response. The table below summarizes them, and specifies the CBOR key to use instead of the full descriptive name.

Note that the media type application/ace-groupcomm+cbor MUST be used when these fields are transported.


 Name         | CBOR Key | CBOR Type     |   Reference
--------------|----------|---------------|---------------
 error         |   TBD    | int   | {{key-distr}}
 error_description |   TBD    | text string   | {{key-distr}}
 scope        |   TBD    | byte string   | {{gid-post}}
 get_pub_keys |   TBD    | array / simple value null  | {{gid-post}}, {{pubkey-fetch}}
 client_cred  |   TBD    | byte string   | {{gid-post}}
 cnonce       |   TBD    | byte string   | {{gid-post}}
 client_cred_verify |   TBD    | byte string   | {{gid-post}}
 pub_keys_repos   |   TBD    | text string         | {{gid-post}}
 control_uri | TBD | text string | {{gid-post}}
 gkty          |   TBD    | integer / text string   | {{gid-post}}
 key          |   TBD    | see "ACE Groupcomm Key Types" Registry     | {{gid-post}}
 num          |   TBD    | int           | {{gid-post}}
 ace-groupcomm-profile |   TBD    | int           | {{gid-post}}
 exp          |   TBD    | int           | {{gid-post}}
 pub_keys     |   TBD    | array   | {{gid-post}}
 peer_roles     |   TBD    | array   | {{gid-post}}
 peer_identifiers     |   TBD    | array   | {{gid-post}}
 group_policies      |   TBD    | map           | {{gid-post}}
 mgt_key_material    |   TBD    | byte string   | {{gid-post}}
 sign_info | TBD | array | {{gid-post}}
 gid          |   TBD    | array   | {{ace-group-fetch}}
 gname        |   TBD    | array of text strings        | {{ace-group-fetch}}
 guri         |   TBD    | array of text strings   | {{ace-group-fetch}}
 kdcchallenge | TBD | byte string | {{node-pub-key-post}}

# ACE Groupcomm Error Identifiers {#error-types}

This specification defines a number of values that the KDC can include as error identifiers, in the 'error' field of an error response with Content-Format application/ace-groupcomm+cbor.

 Value        | Description |
--------------|-------------|
 0            | Operation permitted only to group members |
 1            | Request inconsistent with the current roles |
 2            | Public key incompatible with the group configuration |
 3            | Invalid proof-of-possession evidence |
 4            | No available node identifiers |
 5            | Group membership terminated |
 6            | Group deleted |

# Security Considerations {#sec-cons}

When a Client receives a message from a sender for the first time, it needs to have a mechanism in place to avoid replay, e.g., Appendix B.2 of {{RFC8613}}. In case the Client rebooted and lost the security state used to protect previous communication with that sender, such a mechanism is useful for the recipient to be on the safe side.

Besides, if the KDC has renewed the group keying material, and the time interval between the end of the rekeying process and the joining of the Client is sufficiently small, that Client is also on the safe side, since replayed older messages protected with the previous keying material will not be accepted.

The KDC must renew the group keying material upon its expiration.

The KDC should renew the keying material upon group membership change, and should provide it to the current group members through the rekeying scheme used in the group.

The KDC should renew the group keying material after rebooting, even in the case where all keying material is stored in persistent storage. However, if the KDC relies on Observe responses to notify the group of renewed keying material, after rebooting the KDC will have lost all the current ongoing Observations with the group members, and the previous keying material will be used to protect messages in the group anyway. The KDC will rely on each node requesting updates of the group keying material to establish the new keying material in the nodes, or, if implemented, it can push the update to the nodes in the group using the 'control_uri' resource.

The KDC may enforce a rekeying policy that takes into account the overall time required to rekey the group, as well as the expected rate of changes in the group membership.

That is, the KDC may not rekey the group at every membership change, for instance if members' joining and leaving occur frequently and performing a group rekeying takes too long.
The KDC may rekey the group after a minimum number of group members have joined or left within a given time interval, or after maximum amount of time since the last rekeying was completed, or yet during predictable network inactivity periods.

However, this would result in the KDC not constantly preserving backward and forward security. Newly joining group members could be able to access the keying material used before their joining, and thus could access past group communications. Also, until the KDC performs a group rekeying, the newly leaving nodes would still be able to access upcoming group communications that are protected with the keying material that has not yet been updated.

The KDC needs to have a mechanism in place to detect DoS attacks from nodes constantly initiating rekey events (for example by updating their public key), such as removing these nodes from the group.

The KDC also needs to have a congestion control mechanism in place to avoid network congestion when the KDC renews the group keying material; CoAP and Observe give guidance on such mechanisms, see {{Section 4.7 of RFC7252}} and {{Section 4.5.1 of RFC7641}}.


## Update of Keying Material

A group member can receive a message shortly after the group has been rekeyed, and new keying material has been distributed by the KDC. In the following two cases, this may result in misaligned keying material between the group members.

In the first case, the sender protects a message using the old keying material. However, the recipient receives the message after having received the new keying material, hence not being able to correctly process it. A possible way to ameliorate this issue is to preserve the old, recent, keying material for a maximum amount of time defined by the application. By doing so, the recipient can still try to process the received message using the old retained keying material. Note that a former (compromised) group member can take advantage of this by sending messages protected with the old retained keying material. Therefore, a conservative application policy should not admit the storage of old keying material.

In the second case, the sender protects a message using the new keying material, but the recipient receives that request before having received the new keying material. Therefore, the recipient would not be able to correctly process the request and hence discards it. If the recipient receives the new keying material shortly after that and the application at the sender endpoint performs retransmissions, the former will still be able to receive and correctly process the message. In any case, the recipient should actively ask the KDC for an updated keying material according to an application-defined policy, for instance after a given number of unsuccessfully decrypted incoming messages.

A node that has left the group should not expect any of its outgoing messages to be successfully processed, if received after its leaving, due to a possible group rekeying occurred before the message reception.

## Block-Wise Considerations

If the block-wise options {{RFC7959}} are used, and the keying material is updated in the middle of a block-wise transfer, the sender of the blocks just changes the keying material to the updated one and continues the transfer. As long as both sides get the new keying material, updating the keying material in the middle of a transfer will not cause any issue. Otherwise, the sender will have to transmit the message again, when receiving an error message from the recipient.

Compared to a scenario where the transfer does not use block-wise, depending on how fast the keying material is changed, the nodes might consume a larger amount of the network bandwidth resending the blocks again and again, which might be problematic.

# IANA Considerations

This document has the following actions for IANA.

## Media Type Registrations {#media-type}

   This specification registers the 'application/ace-groupcomm+cbor' media type for messages of the protocols defined in this document following the ACE exchange and carrying parameters encoded in CBOR. This registration follows the procedures specified in {{RFC6838}}.

   Type name: application

   Subtype name: ace-groupcomm+cbor

   Required parameters: N/A

   Optional parameters: N/A

   Encoding considerations: Must be encoded as CBOR map containing the protocol parameters defined in \[this document\].

   Security considerations: See {{sec-cons}} of this document.

   Interoperability considerations: n/a

   Published specification: \[this document\]

   Applications that use this media type: The type is used by authorization servers, clients and resource servers that support the ACE groupcomm framework as specified in \[this document\].

   Fragment identifier considerations: N/A
   
   Additional information: N/A

   Person & email address to contact for further information:
   <iesg@ietf.org>

   Intended usage: COMMON

   Restrictions on usage: None

   Author: Francesca Palombini <francesca.palombini@ericsson.com>

   Change controller: IESG

## CoAP Content-Formats {#content-type}

This specification registers the following entry to the "CoAP Content-Formats" registry within the "CoRE Parameters" registry group:

Media Type: application/ace-groupcomm+cbor

Encoding: -

ID: TBD

Reference: \[this document\]

## OAuth Parameters {#iana-kinfo}

   The following registrations are done for the OAuth Parameters Registry following the procedure specified in {{Section 11.2 of RFC6749}}:

*  Parameter name: sign_info
*  Parameter usage location: client-rs request, rs-client response
*  Change Controller: IESG
*  Specification Document(s): \[\[This specification\]\]

&nbsp;

*  Parameter name: kdcchallenge
*  Parameter usage location: rs-client response
*  Change Controller: IESG
*  Specification Document(s): \[\[This specification\]\]

## OAuth Parameters CBOR Mappings {#iana-kinfo-map}

The following registrations are done for the OAuth Parameters CBOR
Mappings Registry following the procedure specified in {{Section 8.10 of I-D.ietf-ace-oauth-authz}}:

* Name: sign_info
* CBOR Key: TBD (range -256 to 255)
* Value Type: Simple value null / array
* Reference: \[\[This specification\]\]

&nbsp;

* Name: kdcchallenge
* CBOR Key: TBD (range -256 to 255)
* Value Type: Byte string
* Reference: \[\[This specification\]\]

## ACE Groupcomm Parameters {#iana-reg}

This specification establishes the "ACE Groupcomm Parameters" IANA Registry. The
Registry has been created to use the "Expert Review" registration procedure {{RFC8126}}. Expert review guidelines are provided in {{review}}.

The columns of this Registry are:

* Name: This is a descriptive name that enables easier reference to
  the item. The name MUST be unique. It is not used in the encoding.

* CBOR Key: This is the value used as CBOR key of the item. These values MUST be unique. The value can be a positive integer, a negative integer, or a string.

* CBOR Type: This contains the CBOR type of the item, or a pointer to the registry that defines its type, when that depends on another item.

* Reference: This contains a pointer to the public specification for the item.

This Registry has been initially populated by the values in {{params}}. The Reference column for all of these entries refers to sections of this document.

## ACE Groupcomm Key Types {#iana-key}

This specification establishes the "ACE Groupcomm Key Types" IANA Registry. The
Registry has been created to use the "Expert Review" registration procedure {{RFC8126}}. Expert review guidelines are provided in {{review}}.

The columns of this Registry are:

* Name: This is a descriptive name that enables easier reference to
  the item. The name MUST be unique. It is not used in the encoding.

* Key Type Value: This is the value used to identify the keying material. These values MUST be unique.  The value can be a positive integer, a negative integer, or a text string.

* Profile: This field may contain one or more descriptive strings of application profiles to be used with this item. The values should be taken from the Name column of the "ACE Groupcomm Profiles" Registry.

* Description: This field contains a brief description of the keying material.

* References: This contains a pointer to the public specification for the format of the keying material, if one exists.

This Registry has been initially populated by the values in {{gkty}}. The specification column for all of these entries will be this document.

## ACE Groupcomm Profiles

This specification establishes the "ACE Groupcomm Profiles" IANA Registry. The Registry has been created to use the "Expert Review" registration procedure {{RFC8126}}. Expert review guidelines are provided in {{review}}. It should be noted that, in addition to the expert review, some portions of the Registry require a specification, potentially a Standards Track RFC, to be supplied as well.

The columns of this Registry are:

* Name: The name of the application profile, to be used as value of the profile attribute.

* Description: Text giving an overview of the application profile and the context it is developed for.

* CBOR Value: CBOR abbreviation for the name of this application profile. Different ranges of values use different registration policies [RFC8126]. Integer values from -256 to 255 are designated as Standards Action. Integer values from -65536 to -257 and from 256 to 65535 are designated as Specification Required. Integer values greater than 65535 are designated as Expert Review. Integer values less than -65536 are marked as Private Use.

* Reference: This contains a pointer to the public specification of the abbreviation for this application profile, if one exists.

## ACE Groupcomm Policies

This specification establishes the "ACE Groupcomm Policies" IANA Registry. The Registry has been created to use the "Expert Review" registration procedure {{RFC8126}}. Expert review guidelines are provided in {{review}}. It should be noted that, in addition to the expert review, some portions of the Registry require a specification, potentially a Standards Track RFC, to be supplied as well.

The columns of this Registry are:

* Name: The name of the group communication policy.

* CBOR label: The value to be used to identify this group communication policy.  Key map labels MUST be unique. The label can be a positive integer, a negative integer or a string.  Integer values between 0 and 255 and strings of length 1 are designated as Standards Track Document required. Integer values from 256 to 65535 and strings of length 2 are designated as Specification Required.  Integer values greater than 65535 and strings of length greater than 2 are designated as expert review.  Integer values less than -65536 are marked as private use.

* CBOR type: the CBOR type used to encode the value of this group communication policy.

* Description: This field contains a brief description for this group communication policy.

* Reference: This field contains a pointer to the public specification providing the format of the group communication policy, if one exists.

This registry will be initially populated by the values in {{fig-ACE-Groupcomm-Policies}}.

## Sequence Number Synchronization Methods

This specification establishes the "Sequence Number Synchronization Methods" IANA Registry. The Registry has been created to use the "Expert Review" registration procedure {{RFC8126}}. Expert review guidelines are provided in {{review}}. It should be noted that, in addition to the expert review, some portions of the Registry require a specification, potentially a Standards Track RFC, to be supplied as well.

The columns of this Registry are:

* Name: The name of the sequence number synchronization method.

* Value: The value to be used to identify this sequence number synchronization method.

* Description: This field contains a brief description for this sequence number synchronization method.

* Reference: This field contains a pointer to the public specification describing the sequence number synchronization method.

## Interface Description (if=) Link Target Attribute Values {#if-ace-group}

This specification registers the following entry to the "Interface Description (if=) Link Target Attribute Values Registry" registry within the "CoRE Parameters" registry group:

* Attribute Value: ace.group

* Description: The 'ace group' interface is used to provision keying material and related information and policies to members of a group using the Ace framework.

* Reference: \[This Document\]

## CBOR Tags {#iana-cbor-tags}

This specification registers the following entry to the "CBOR Tags" registry:

* Tag : TBD_TAG

* Data Item: byte string

* Semantics: Extended ACE scope format, including the identifier of the used scope semantics.

* Reference: \[This Document\]

## ACE Scope Semantics {#iana-scope-semantics}

This specification establishes the "ACE Scope Semantics" IANA Registry. The Registry has been created to use the "Expert Review" registration procedure {{RFC8126}}. Expert review guidelines are provided in {{review}}. It should be noted that, in addition to the expert review, some portions of the Registry require a specification, potentially a Standards Track RFC, to be supplied as well.

The columns of this Registry are:

* Value: The value to be used to identify this scope semantics. The value MUST be unique. The value can be a positive integer or a negative integer. Integer values between 0 and 255 are designated as Standards Track Document required. Integer values from 256 to 65535 are designated as Specification Required. Integer values greater than 65535 are designated as expert review. Integer values less than -65536 are marked as private use.

* Description: This field contains a brief description of the scope semantics.

* Reference: This field contains a pointer to the public specification defining the scope semantics, if one exists.

## ACE Groupcomm Errors {#iana-ace-groupcomm-errors}

This specification establishes the "ACE Groupcomm Errors" IANA Registry. The Registry has been created to use the "Expert Review" registration procedure {{RFC8126}}. Expert review guidelines are provided in {{review}}. It should be noted that, in addition to the expert review, some portions of the Registry require a specification, potentially a Standards Track RFC, to be supplied as well.

The columns of this Registry are:

* Value: The value to be used to identify the error. The value MUST be unique. The value can be a positive integer or a negative integer. Integer values between 0 and 255 are designated as Standards Track Document required. Integer values from 256 to 65535 are designated as Specification Required. Integer values greater than 65535 are designated as expert review. Integer values less than -65536 are marked as private use.

* Description: This field contains a brief description of the error.

* Reference: This field contains a pointer to the public specification defining the error, if one exists.

This Registry has been initially populated by the values in {{error-types}}. The Reference column for all of these entries refers to this document.

## Expert Review Instructions {#review}

The IANA Registries established in this document are defined as expert review.
This section gives some general guidelines for what the experts should be looking for, but they are being designated as experts for a reason so they should be given substantial latitude.

Expert reviewers should take into consideration the following points:

* Point squatting should be discouraged.
 Reviewers are encouraged to get sufficient information for registration requests to ensure that the usage is not going to duplicate one that is already registered and that the point is likely to be used in deployments.
 The zones tagged as private use are intended for testing purposes and closed environments, code points in other ranges should not be assigned for testing.

* Specifications are required for the standards track range of point assignment.
 Specifications should exist for specification required ranges, but early assignment before a specification is available is considered to be permissible.
 Specifications are needed for the first-come, first-serve range if they are expected to be used outside of closed environments in an interoperable way.
 When specifications are not provided, the description provided needs to have sufficient information to identify what the point is being used for.

* Experts should take into account the expected usage of fields when approving point assignment.
 The fact that there is a range for standards track documents does not mean that a standards track document cannot have points assigned outside of that range.
 The length of the encoded value should be weighed against how many code points of that length are left, the size of device it will be used on, and the number of code points left that encode to that size.

--- back

# Requirements on Application Profiles {#req}

This section lists the requirements on application profiles of this specification,for the convenience of application profile designers.

* REQ1: If the value of the GROUPNAME URI path and the group name in the access token scope (gname in {{ssec-authorization-response}}) don't match, specify the mechanism to map the GROUPNAME value in the URI to the group name (REQ1) (see {{kdc-if}}).

* REQ2: Specify the encoding and value of roles, for scope entries of 'scope' (see {{ssec-authorization-request}}).

* REQ3: If used, specify the acceptable values for 'sign_alg' (see {{token-post}}).

* REQ4: If used, specify the acceptable values for 'sign_parameters' (see {{token-post}}).

* REQ5: If used, specify the acceptable values for 'sign_key_parameters' (see {{token-post}}).

* REQ6: Specify the acceptable formats for encoding public keys and, if used, the acceptable values for 'pub_key_enc' (see {{token-post}}).

* REQ7: Register a Resource Type for the root url-path, which is used to discover the correct url to access at the KDC (see {{kdc-if}}).

* REQ8: Define what operations (e.g., CoAP methods) are allowed on each resource, for each role defined in REQ2 (see {{token-post}}).


* REQ9: Specify the exact encoding of group identifier (see {{ace-group-fetch}}).

* REQ10: Specify the exact format of the 'key' value (see {{gid-post}}).

* REQ11: Specify the acceptable values of 'gkty' (see {{gid-post}}).

* REQ12: Specify the format of the identifiers of group members (see {{gid-post}}).

* REQ13: Specify the communication protocol the members of the group must use (e.g., multicast CoAP).

* REQ14: Specify the security protocol the group members must use to protect their communication (e.g., group OSCORE). This must provide encryption, integrity and replay protection.

* REQ15: Specify and register the application profile identifier (see {{gid-post}}).

* REQ16: Specify policies at the KDC to handle ids that are not included in 'get_pub_keys' (see {{pubkey-fetch}}).

* REQ17: If used, specify the format and content of 'group\_policies' and its entries. Specify the policies default values (see {{gid-post}}).

* REQ18: Specify the format of newly-generated individual keying material for group members, or of the information to derive it, and corresponding CBOR label (see {{node-get}}).

* REQ19: Specify how the communication is secured between Client and KDC. Optionally, specify transport profile of ACE {{I-D.ietf-ace-oauth-authz}} to use between Client and KDC (see {{ssec-key-distribution-exchange}}.

* REQ20: Specify the exact approaches used to compute and verify the PoP evidence to include in 'client_cred_verify', and which of those approaches is used in which case (see {{gid-post}}).

* REQ21: Specify how the nonce N_S is generated, if the token is not provided to the KDC through the Token Transfer Request to the authz-info endpoint (e.g., if it is used directly to validate TLS instead).

* REQ22: Specify if 'mgt\_key\_material' used, and if yes specify its format and content (see {{gid-post}}). If the usage of ‘mgt_key_material’ is indicated and its format defined for a specific key management scheme, that format must explicitly indicate the key management scheme itself. If a new rekeying scheme is defined to be used for an existing ‘mgt_key_material’ in an existing profile, then that profile will have to be updated accordingly, especially with respect to the usage of ‘mgt_key_material’ related format and content.

* REQ23: Define the initial value of the 'num' parameter (see {{gid-post}}).

* REQ24: Specify and register the identifier of newly defined semantics for binary scopes (see {{sec-extended-scope}}).

* REQ25: Specify if any functionality of the KDC interface as defined in this document is not supported (see {{kdc-if}}).

* OPT1: Optionally, specify the negotiation of parameter values for signature algorithm and signature keys, if 'sign_info' is not used (see {{token-post}}).

* OPT2: Optionally, specify the additional parameters used in the exchange of Token Transfer Request and Response (see {{token-post}}).

* OPT3: Optionally, specify the encoding of 'pub\_keys\_repos' if the default is not used (see {{gid-post}}).

* OPT4: Optionally, specify policies that instruct clients to retain messages and for how long, if they are unsuccessfully decrypted (see {{update-keys}}). This makes it possible to decrypt such messages after getting updated keying material.

* OPT5: Optionally, specify possible or required payload formats for specific error cases.

* OPT6: Optionally, specify the behavior of the handler in case of failure to retrieve a public key for the specific node (see {{gid-post}}).

* OPT7: Optionally, specify CBOR values to use for abbreviating identifiers of roles in the group or topic (see {{ssec-authorization-request}}).

* OPT8: Optionally, specify for the KDC to perform group rekeying (together or instead of renewing individual keying material) when receiving a Key Renewal Request (see {{new-keys}}).

* OPT9: Optionally, specify the functionalities implemented at the 'control_uri' resource hosted at the Client, including message exchange encoding and other details (see {{gid-post}}).

* OPT10: Optionally, specify how the identifier of the sender's public key is included in the group request (see {{update-pub-key}}).
   
* OPT11: Optionally, specify additional identifiers of error types, as values of the 'error' field in an error response from the KDC.

# Extensibility for Future COSE Algorithms # {#sec-future-cose-algs}

As defined in {{Section 8.1 of I-D.ietf-cose-rfc8152bis-algs}}, future algorithms can be registered in the "COSE Algorithms" Registry {{COSE.Algorithms}} as specifying none or multiple COSE capabilities.

To enable the seamless use of such future registered algorithms, this section defines a general, agile format for each 'sign_info_entry' of the 'sign_info' parameter in the Token Transfer Response, see {{sign-info}}.

If any of the currently registered COSE algorithms is considered, using this general format yields the same structure of 'sign_info_entry' defined in this document, thus ensuring retro-compatibility.

## Format of 'sign_info_entry' ## {#sec-future-cose-algs-sign-info-entry}

The format of each 'sign_info_entry' (see {{sign-info}}) is generalized as follows. Given N the number of elements of the 'sign_parameters' array, i.e., the number of COSE capabilities of the signature algorithm, then:

* 'sign_key_parameters' is replaced by N elements 'sign_capab_i', each of which is a CBOR array.

* The i-th array following 'sign_parameters', i.e., 'sign_capab_i' (i = 0, ..., N-1), is the array of COSE capabilities for the algorithm capability specified in 'sign_parameters'\[i\].

~~~~~~~~~~~ CDDL
sign_info_entry =
[
  id : gname / [ + gname ],
  sign_alg : int / tstr,
  sign_parameters : [ alg_capab_1 : any,
                      alg_capab_2 : any,
                      ...,
                      alg_capab_N : any],
  sign_capab_1 : [ any ],
  sign_capab_2 : [ any ],
  ...,
  sign_capab_N : [ any ],
  pub_key_enc = int / nil
]

gname = tstr
~~~~~~~~~~~
{: #fig-sign-info-entry-general title="'sign_info_entry' with general format"}

# Document Updates # {#sec-document-updates}

RFC EDITOR: PLEASE REMOVE THIS SECTION.

## Version -04 to -05 ## {#sec-04-05}

* Updated uppercase/lowercase URI segments for KDC resources.

* Supporting single Access Token for multiple groups/topics.

* Added 'control_uri' parameter in the Joining Request.

* Added 'peer_roles' parameter to support legal requesters/responders.

* Clarification on stopping using owned keying material.

* Clarification on different reasons for processing failures, related policies, and requirement OPT4.

* Added a KDC sub-resource for group members to upload a new public key.

* Possible group rekeying following an individual Key Renewal Request.

* Clarified meaning of requirement REQ3; added requirement OPT8.

* Editorial improvements.

## Version -03 to -04 ## {#sec-03-04}

* Revised RESTful interface, as to methods and parameters.

* Extended processing of joining request, as to check/retrieval of public keys.

* Revised and extended profile requirements.

* Clarified specific usage of parameters related to signature algorithms/keys.

* Included general content previously in draft-ietf-ace-key-groupcomm-oscore

* Registration of media type and content format application/ace-group+cbor

* Editorial improvements.

## Version -02 to -03 ## {#sec-02-03}

* Exchange of information on the signature algorithm and related parameters, during the Token POST (Section 3.3).

* Restructured KDC interface, with new possible operations (Section 4).

* Client PoP signature for the Joining Request upon joining (Section 4.1.2.1).

* Revised text on group member removal (Section 5).

* Added more profile requirements (Appendix A).

## Version -01 to -02 ## {#sec-01-02}

* Editorial fixes.

* Distinction between transport profile and application profile (Section 1.1).

* New parameters 'sign_info' and 'pub_key_enc' to negotiate parameter values for signature algorithm and signature keys (Section 3.3).

* New parameter 'type' to distinguish different Key Distribution Request messages (Section 4.1).

* New parameter 'client_cred_verify' in the Key Distribution Request to convey a Client signature (Section 4.1).

* Encoding of 'pub_keys_repos' (Section 4.1).

* Encoding of 'mgt_key_material' (Section 4.1).

* Improved description on retrieval of new or updated keying material (Section 6).

* Encoding of 'get_pub_keys' in Public Key Request (Section 7.1).

* Extended security considerations (Sections 10.1 and 10.2).

* New "ACE Public Key Encoding" IANA Registry (Section 11.2).

* New "ACE Groupcomm Parameters" IANA Registry (Section 11.3), populated with the entries in Section 8.

* New "Ace Groupcomm Request Type" IANA Registry (Section 11.4), populated with the values in Section 9.

* New "ACE Groupcomm Policy" IANA Registry (Section 11.7) populated with two entries "Sequence Number Synchronization Method" and "Key Update Check Interval" (Section 4.2).

* Improved list of requirements for application profiles (Appendix A).

## Version -00 to -01 ## {#sec-00-01}

* Changed name of 'req_aud' to 'audience' in the Authorization Request (Section 3.1).

* Defined error handling on the KDC (Sections 4.2 and 6.2).

* Updated format of the Key Distribution Response as a whole (Section 4.2).

* Generalized format of 'pub_keys' in the Key Distribution Response (Section 4.2).

* Defined format for the message to request leaving the group (Section 5.2).

* Renewal of individual keying material and methods for group rekeying initiated by the KDC (Section 6).

* CBOR type for node identifiers in 'get_pub_keys' (Section 7.1).

* Added section on parameter identifiers and their CBOR keys (Section 8).

* Added request types for requests to a Join Response (Section 9).

* Extended security considerations (Section 10).

* New IANA registries "ACE Groupcomm Key Registry", "ACE Groupcomm Profile Registry", "ACE Groupcomm Policy Registry" and "Sequence Number Synchronization Method Registry" (Section 11).

* Added appendix about requirements for application profiles of ACE on group communication (Appendix A).

# Acknowledgments
{: numbered="no"}

The following individuals were helpful in shaping this document: Christian Amsuess, Carsten Bormann, Rikard Hoeglund, Ben Kaduk, Watson Ladd, John Mattsson, Daniel Migault, Jim Schaad, Ludwig Seitz, Goeran Selander, Cigdem Sengul and Peter van der Stok.

The work on this document has been partly supported by VINNOVA and the Celtic-Next project CRITISEC; by the H2020 project SIFIS-Home (Grant agreement 952652); and by the EIT-Digital High Impact Initiative ACTIVE.

--- fluff
