NIP-X
=====

Key Migration and Revocation
------

`draft` `optional` `author:braydonf`

This NIP defines the protocol that SHOULD BE implemented by clients and relays to handle a key migration and revocation of a key. This specification uses one or multiple methods to verify the successor key, if specified, including a social graph, backup recovery keys and side-channel identity anchor points.

## Backup Keys

### Backup Keys Setup Event

This is an event that is non-replaceable and ONLY ONE can be considered valid. Clients SHOULD implement a means to verify that users are aware of these conditions to avoid accidental broadcast of the event. Every client is individually responsible for storing a copy of this event and SHOULD also store it on relays, signed via attestations. Any future events of this type MUST NOT be replaced automatically as it could be from an attacker due to a compromised or stolen private key. Clients MAY provide a manual verification process that can be verified through a side-channel to be able to independently replace the setup event. The recovery setup event can assign anywhere from 1 to `n` backup keys assigned to be able to sign the change and revocation event. A threshold number of keys (`m` of `n`) can be assigned to verify this event.

```js
{
  "kind": <x>,
  "pubkey": "<user-pubkey>",
  "tags": [
    ["p", "<backup-pubkey-1>"],
    ["p", "<backup-pubkey-2>"],
    ["p", "<backup-pubkey-3>"],
    ["threshold", "2"],
    ["backup-key-setup"]
  ],
  "content": "",
  // ...
}
```

* A `p` tag MUST BE included and be the hex encoded public keys for the backup recovery keys.
* If `threshold` tag is NOT included, the threshold should default to `1`.
* The `backup-key-setup` MUST be included, it's otherwise ignored, however can help prevent making this event by chance or accident.
* The content MAY include with a comment, most clients can ignore this field.
* Relays MAY store multiple events from a pubkey of this kind.
* Clients MUST only consider one to be valid. If multiple exist, a user interface SHOULD provide a means to pick one. There can be various means to help select the key including; sending a DM, displaying public attestations from within a social graph or NIP-03 (OpenTimestamps Attestations for Events) timestamps associated with the event.

### Backup Keys Social Attestation Event

This is an event that is non-replaceable. This is a means to select a valid and verified Backup Keys Setup Event for another pubkey. The attestation can be either private or public. By making a public attestation, others in the network can see that you're able to verify the backup key for a profile; this can help built a robust fault-tolerant social graph. The default option SHOULD be private. The primary purpose for this event is for clients to be able to verify the backup keys for a Key Migration and Revocation event (see below).

```js
{
  "kind": <x>,
  "pubkey": "<user-pubkey>",
  "tags": [
    ["p", "<pubkey-of-friend>"],
    ["backup-key-attestation"]
  ],
  "content": "<backup-key-setup-event>",
  // ...
}
```

* The `p` tag for a public attestation MUST include a public key and MUST match the pubkey for the Backup Key Setup Event. A private attestation MUST NOT include this tag.
* The `content` field MUST include the Backup Key Setup Event either encrypted and private or unencrypted and public.
* The `backup-key-attestation` MUST be included, it's otherwise ignored, however can help prevent making this event by chance or accident.

## Key Migration and Revocation

### Event

This is an event that is non-replaceable. Clients SHOULD implement a means to verify that users are aware of these conditions to avoid accidental broadcast of the event.

A client when receiving the event SHOULD present the user with a user interface to be able to verify the authenticity of the request, independently, and the provide a way to unfollow the identity and follow the new identity instead. A client MAY keep a record of the previous key for reference. The existence of a Backup Keys Setup Event is NOT REQUIRED to be able to handle a Key Migration and Revocation Event, however it will not be able to use any backup recovery keys to verify and instead will only rely on the other two methods to verify.

```js
{
  "kind": <x>,
  "pubkey": "<user-pubkey>",
  "tags": [
    ["new-key", "<new-key>"],
    ["e", "<event-id-of-backup-key-setup>"],
    ["key-migration-and-revocation"]
  ],
  "content": "<backup-key-signatures-and-optional-comment>",
  // ...
}
```

* The `new-key` tag SHOULD BE included, otherwise it will be a key revocation without a successor key and all events will be deleted.
* The `content` MUST include the signatures for `m` of `n` public keys as specified in the referenced Backup Keys Setup Event. The signature should be over the entire event as similar to the event signature.
* If a `new-key` IS provided, the `key-migration-and-revocation` tag MUST be included, it's otherwise ignored, however can help prevent making this event by chance or accident.
* If a `new-key` IS NOT provided, the `key-revocation` tag MUST be included, this is also to help prevent mistakes.

### Event Verification for Clients

There are several ways to be able to verify the authenticity of a Key Migration and Revocation Event. The primary one is through the use of the defined backup recovery keys, the second means is by trusting those whom have already verified the change that are in a social graph and are now following the new key, the third way is through the use of side-channel external identities. All of these methods SHOULD be implemented, this specification only REQUIRES that at-least one of them is implemented. A client MUST NOT automatically, without user interaction, verify and switch to a new key and SHOULD present to the user a way to verify and accept the change. Please see External Refereces (below) for additional guidance for UX design.

#### Behaviors
- Verification of the event and NOT the backup keys determines if a revocation is valid.
- Upon a valid key revocation:
  - All events MUST display a warning that the event is from a revoked pubkey.
  - All events of a revoked pubkey SHOULD be deleted locally after 60 days.
- A user interface MUST display to the user an option to accept or reject a key migration. This SHOULD include the backup keys and how many have valid signatures, social graph verification of those that they follow that are now following the new key, and an other side-channel pinned identities such as with NIP-05. At least one method MUST be provided.
- Upon a valid key migration:
  - The old key MUST be unfollowed and the new key MUST be followed.
  - The old key MUST be stored for future reference to able to block and delete future events from the old key.
  - The old key MAY be added to other mute lists.

### Event Verification for Relays

Relays MUST delete all events after receiving and verifying a Key Migration and Revocation Event. For relays, the event MUST be verified by the user key AND NOT the backup keys. A relay MUST not accept any further events from the key EXCEPT additional key change and revocation events; this is to ensure that if a key is compromised and a fraudulent event is made, an honest event can also be made and broadcast. Each client can then verify which is honest.

#### Behaviors
- Verification of the event and NOT the backup keys determines if a revocation is valid.
- Upon a valid key revocation:
  - All events of a revoked pubkey MUST be deleted after 60 days.
  - All future events, as determined when it was received and not the date on the event, of a revoked pubkey MUST be rejected except for another Key Migration and Revocation event.
- Key migration is handled only on the client and the backup keys do not need to be verified.

## Rationale

## Acknowledgments

## External References
