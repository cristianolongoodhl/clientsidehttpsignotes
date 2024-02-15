## Activity delivery in ActivityPub
As reported in the [ActivityPub specification](https://www.w3.org/TR/2018/REC-activitypub-20180123/), an ActivityPub server must provide each [`Actor`](https://www.w3.org/TR/activitypub/#actor-objects) with two ['Collection's](https://www.w3.org/TR/activitystreams-vocabulary/#dfn-collection): [`outbox`](https://www.w3.org/TR/activitypub/#outbox) and [`inbox`](https://www.w3.org/TR/activitypub/#inbox). In addition, the addresses of these collections must be listed in the publicly available `Actor` description. 

Let us call _target actors_ of an `Activity` all the `Actor`s to which the `Activity` has to be delivered. Target actors of an activity are specified in the activity itself as described in the specification at [6.1 Client Addressing](https://www.w3.org/TR/activitypub/#client-addressing).

Now let use recall the [Activity](https://www.w3.org/TR/activitystreams-vocabulary/#dfn-activity) delivery process in ActivityPub. 

- Client to Server
  * <span id="#csa">[CSA]</span> In order to deliver an activity to activity target actors the (user corresponding to) sender `Actor` has to place the activity into the outbox the server provided to it.
  * <span id="#cso">[CSO]</span> Otherwise, the `Actor` may send to her server an [`Object`](https://www.w3.org/TR/activitystreams-vocabulary/#dfn-object) which cannot be recognized as an `Activity` (see [Note 1](#note1)). This case must be handled by the server, as described in the specification at [6.2.1 Object creation without a Create Activity](https://www.w3.org/TR/activitypub/#object-without-create), with generating a novel `Create` activity _wrapping_ the object sent by the sender, and placing this novel activity in the outbox.
- Server to Server
  * <span id="#cso">[SS]</span> once that the `Activity` has been placed into the sender outbox, the server sends it to the `inbox`es of all the activity target actors. Notice that some of these inboxes may reside outside the sender server. In these cases we talk about a _Server to Server_ communication, which is described in the specification at [7.1 Delivery](https://www.w3.org/TR/activitypub/#delivery).
 
![Activity delivery diagram](/clientsidehttpsignotes/docs/assets/delivery.svg)
 
Notice that security mechanisms for both client-to-server and server-to-server communications are not specified in the protocol, as they are left to implementations. The ActivityPub specification refers (in a non-mandatory way) to the [Social Web Community Group Authentication and Authorization best practices report](https://www.w3.org/wiki/SocialCG/ActivityPub/Authentication_Authorization), which effectivelly guided the most part of the ActivityPub implementations currently availabe.

For client to server comunications, the report suggests [OAuth 2](https://oauth.net/2/), which is a well-established authentication and authorization protocol wich, in addition, provides confidentiality features.

For server to server comunications, the approach proposed by the report is to sign messages with [HTTP Signatures](https://tools.ietf.org/html/draft-cavage-http-signatures-08) as follows:
- of course, the headers taken into consideration by the signature must contains at minimun the message content _digest_, see also [Note 2](#note2);
- the message must be signed with a key associated to the sender actor;
- the `keyId` field of the `Signature` header should link to the `Actor` description which, in turn, must provide the public part of the actor key via a `publicKey` property.

Then, the receiving server can check message integrity with fetching the actor public key through `keyId` and checking the message body (digest) against the public key provided in the actor description.

### <span id="note1">Note 1</span>
This part of ActivityPub is quite controversial as in ActivityPub every `Activity` is an `Object`, so that one cannot deduce that an `Object` is not an `Activity` if not explicitly stated.     

### <span id="note2">Note 2</span>
Some implementations such as, for example, [Mastodon](https://joinmastodon.org/) require to take into consideration also the `date` header in the signature, in order to handle and eventually discard old messages which may be sent by malitious servers.  

## HTTP Signature in Mastodon

As reported in [How to implement a basic ActivityPub server](https://blog.joinmastodon.org/2018/06/how-to-implement-a-basic-activitypub-server/#http-signatures) (see also [Note 2](#note2)) Mastodon servers defines a 30 seconds validity interval for messages. I.e., mastodon servers will discard messages recevived 30 seconds later the date reported in the `date` header.

A mastodon servers acts as a _key holder_ for users' key pairs: key pairs of users are generated, stored and guarded by the server, which will use them to generate the signature header when delivering activities.

This approach has some drawbacks:

- <span id="sd1">[SD1]</span> first and more relevant, a malitious server may send spurious messages claiming that they must attributed to some of its users;
- <span id="sd1">[SD2]</span> in addition, if the server is compromised, all the private keys of its users are at risk and should be invalidated.



