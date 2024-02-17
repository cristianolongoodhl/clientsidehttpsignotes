## Activity delivery in ActivityPub
As reported in the [ActivityPub specification](https://www.w3.org/TR/2018/REC-activitypub-20180123/), an ActivityPub server must provide each [`Actor`](https://www.w3.org/TR/activitypub/#actor-objects) with two ['Collection's](https://www.w3.org/TR/activitystreams-vocabulary/#dfn-collection): [`outbox`](https://www.w3.org/TR/activitypub/#outbox) and [`inbox`](https://www.w3.org/TR/activitypub/#inbox). In addition, the addresses of these collections must be listed in the publicly available `Actor` description. 

Let us call _target actors_ of an `Activity` all the `Actor`s to which the `Activity` has to be delivered. Target actors of an activity are specified in the activity itself as described in the specification at [6.1 Client Addressing](https://www.w3.org/TR/activitypub/#client-addressing).

Now let use recall the [Activity](https://www.w3.org/TR/activitystreams-vocabulary/#dfn-activity) delivery process in ActivityPub. 

- Client to Server
  * <span id="#csa">[CSA]</span> In order to deliver an activity to activity target actors the (user corresponding to) sender `Actor` has to place the activity into the outbox the server provided to it.
  * <span id="#cso">[CSO]</span> Otherwise, the `Actor` may send to her server an [`Object`](https://www.w3.org/TR/activitystreams-vocabulary/#dfn-object) which cannot be recognized as an `Activity` (see [Note 1](#note1)). This case must be handled by the server, as described in the specification at [6.2.1 Object creation without a Create Activity](https://www.w3.org/TR/activitypub/#object-without-create), with generating a novel `Create` activity _wrapping_ the object sent by the sender, and placing this novel activity in the outbox.
- Server to Server
  * <span id="#cso">[SS]</span> once that the `Activity` has been placed into the sender outbox, the server sends it to the `inbox`es of all the activity target actors. Notice that some of these inboxes may reside outside the sender server. In these cases we talk about a _Server to Server_ communication, which is described in the specification at [7.1 Delivery](https://www.w3.org/TR/activitypub/#delivery).
 
![Activity delivery diagram](docs/assets/delivery.svg)
 
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

![Activity delivery with server side signature diagram](docs/assets/deliveryss.svg)

This approach has some drawbacks:

- <span id="sd1">[SD1]</span> first and more relevant, a malitious server may send spurious messages claiming that they must attributed to some of its users;
- <span id="sd1">[SD2]</span> in addition, if the server is compromised, all the private keys of its users are at risk and should be invalidated;
- <span id="sd1">[SD3]</span> finally, when a user want to migrate to another server, the server she is leaving may be reluctant to provide the private key to the user.

But it has some relevant advantages as well.

- <span id="sa1">[SA1]</span> the client can delegate the automated generation and delivery of some kind of activities, such as for example the `Accept` activity for follow requests;
- <span id="sa2">[SA2]</span> the server can defer the signature generation and the subsequent delivery of the activity under particular circumstances, for example in case overloading which limits the currently availabe resources;
- <span id="sa3">[SA3]</span> in addition, the server can retry the delivery of an activity to a target in box in case of network failures.

## Client-Side signature

Let us now consider a setup where the signature header is generated by the client. In such a setup, the outbox must receive from the client the date and the signature headers, aside with the activity to be sent. Then, the outbox forwards these headers, with in addition the activity digest, to the activity target inboxes, aside with of course the activity.     

![Activity delivery with client side signature diagram](docs/assets/deliverycs.svg)

[Little ActivitiPub Server](https://www.opendatahacklab.org/lap_src) is a proof of concept of such approach, as it provides a javascript-based form to generate the signature and send the required headers and the activity to an outbox.

Obviously, this approach does not require that the server knows client private keys. Instead, it shouldn't so that [[SD1]](#sd1) and [[SD2]](#sd2) cannot hold anymore. More in details: 
- <span id="ca1">[CA1]</span> it has no way to create a spurious message and claim that the message has been produced by one of this clients, as in [[SD1]](#sd1), because it cannot sign messages using client keys;
- <span id="ca2">[CA2]</span> if the server is compromised, as in [[SD2]](#sd2), no client key is compromised, just because the server does not hold private key of any of its clients;
- <span id="ca3">[CA3]</span> when a client migrate to another server, she could provide proofs that the new account she acquired is hold by the same client of the one in the old server by proving that the new account can sign messages with the same key of the previous one.

Aside these advantages, the approaches based on client-side generation has some relevat drawbacks:
- Of course, the sender server cannot provide the functionality recalled in [[CSO]](#cso) as the wrapping activity created by the server has to be signed with the actor key ([Note 3](#note3) is about circumventing this).

### <span id="note3">Note 3</span>
One may argue that the server _should_ sign outgoing activities using the key of its so called [instance actor](https://seb.jambor.dev/posts/understanding-activitypub-part-4-threads/#the-instance-actor), i.e. an actor representing the whole server. This sounds even more appropriate in [[CSO]](#cso), as, in this case, the activity is not provided by the client but generated by the server itself. TODO unfolding
