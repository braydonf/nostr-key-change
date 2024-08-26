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

### Event Handling for Clients

For a client, this event is both a revocation and a migration. The revocation MUST be handled by verifying only the event signature and MUST be automatic. The migration, on the other hand, MUST be presented and verified by the user to accept or reject the migration key change. Please see the External Refereces section for additional guidance for UX design.

#### Behaviors
- Verification of the event and NOT the backup keys determines if a revocation is valid.
- Upon a valid key revocation:
  - All events MUST display a warning that the event is from a revoked pubkey.
  - All events of a revoked pubkey SHOULD be deleted, this MAY BE after a duration of time depending on the client or preferences.
- A user interface MUST display to the user an option to accept or reject a key migration. This SHOULD include the backup keys and how many have valid signatures, social graph verification of those that they follow that are now following the new key, and an other side-channel pinned identities such as with NIP-05. At least one method MUST be provided.
- Upon a valid key migration:
  - The old key MUST be unfollowed and the new key MUST be followed.
  - The old key MUST be stored for future reference to able to block and delete future events from the old key.
  - The old key MAY be added to other mute lists.

### Event Handling for Relays

For a relay, this event is primarily a key revocation, and storing the necessary information for clients to verify the key migration.

#### Behaviors
- Verification of the event and NOT the backup keys determines if a revocation is valid.
- Upon a valid key revocation:
  - All events of a revoked pubkey MUST be deleted. The timeframe that events are deleted MAY BE defined by an agreed upon terms between client and relay.
  - All future events, as determined when it was received and not the date on the event, of a revoked pubkey MUST be rejected except for another Key Migration and Revocation event. This is to ensure that if a key is compromised and a fraudulent event is made, an honest event can also be made and broadcast. Each client can then verify which is honest.
- The backup keys do not need to be verified, all key migration verification is handled by the client.
- For denial-of-service mitigation, a relay MAY require proof-of-work, a small fee or another solution to continue to write Key Migration and Revocation Events. This SHOULD be determined by the terms agreed upon by the client and relay.

## Rationale

## Acknowledgments

## External References
