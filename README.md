## Activity delivery in ActivityPub
As reported in the [ActivityPub specification](https://www.w3.org/TR/2018/REC-activitypub-20180123/), an ActivityPub server must provide each [`Actor`](https://www.w3.org/TR/activitypub/#actor-objects) with two ['Collection's](https://www.w3.org/TR/activitystreams-vocabulary/#dfn-collection): [`outbox`](https://www.w3.org/TR/activitypub/#outbox) and [`inbox`](https://www.w3.org/TR/activitypub/#inbox). In addition, the addresses of these collections must be listed in the publicly available `Actor` description. 

Let us call _target actors_ of an `Activity` all the `Actor`s to which the `Activity` has to be delivered. Target actors of an activity are specified in the activity itself as described in the specification at [6.1 Client Addressing](https://www.w3.org/TR/activitypub/#client-addressing).

Now let use recall the [Activity](https://www.w3.org/TR/activitystreams-vocabulary/#dfn-activity) delivery process in ActivityPub. 

- Client to Server
  * <span id="#csa">[CSA]</span> In order to deliver an activity to activity target actors the (user corresponding to) sender `Actor` has to place the activity into the outbox the server provided to it.
  * <span id="#cso">[CSO]</span> Otherwise, the `Actor` may send to her server an [`Object`](https://www.w3.org/TR/activitystreams-vocabulary/#dfn-object) which cannot be recognized as an `Activity` (see [Note 1](#note1)). This case must be handled by the server, as described in the specification at [6.2.1 Object creation without a Create Activity](https://www.w3.org/TR/activitypub/#object-without-create), with generating a novel `Create` activity _wrapping_ the object send by the server and placing this novel activity in the outbox.
- Server to Server
  * <span id="#cso">[SS]</span> once that the `Activity` has been placed into the sender outbox, the server sends it to the `inbox`es of all the activity target actors. Notice that some of these inboxes may reside outside the sender server. In these cases we talk about a _Server to Server_ communication, which is describer in the specification at Section [7.1 Delivery](https://www.w3.org/TR/activitypub/#delivery).
 
Notice that security mechanisms both for client-to-server and server-to-server communications are not specified in the protocol and they are left to implementations. The ActivityPub specification recalls (in a non-mandatory way) to the [Social Web Community Group Authentication and Authorization best practices report](https://www.w3.org/wiki/SocialCG/ActivityPub/Authentication_Authorization), which effectivelly guided the most part of the ActivityPub implementations currently availabe.

For client to server comunications, the report suggests [OAuth 2](https://oauth.net/2/), which is a well-established authentication and authorization protocolo wich, in addition, provides confidentiality features.

For server to server comunications, the approach proposed by the report is to sign messages with [HTTP Signatures](https://tools.ietf.org/html/draft-cavage-http-signatures-08) as follows:
- of course, the headers taken into consideration by the signature must contains at minimun the message content _digest_;
- the message must be signed with a key associated to the sender actor;
- the `keyId` field of the `Signature` header should link to the `Actor` description which. in turn, must provide the public part of the actor key via a `publicKey`.

Then, the receiving serve can check message integrity with fetching the actor public key indicated in `keyId` and checking the message body (digest) against the public key provided in the actor description.

## <span id="note1">Note 1</span>
This part of ActivityPub is quite controversial as in ActivityPub every `Activity` is an `Object`, so that one cannot deduce that an `Object` is not an `Activity` if not explicitly stated.     
