NIP-X
=====

Key Migration and Revocation
------

`draft` `optional` `author:braydonf`

This NIP defines the protocol that SHOULD BE implemented by clients to handle a key migration and revocation of a key. This specification uses one or multiple methods to verify the successor key, if specified, including a social graph, backup recovery keys and side-channel identity anchor points.

## Backup Keys

### Setup Event

This is an event that is non-replaceable and ONLY ONE can be considered valid. Clients SHOULD implement a means to verify that users are aware of these conditions to avoid accidental broadcast of the event. Every client is individually responsible for storing a copy of this event and SHOULD also store it on relays, signed via attestations. Any future events of this type MUST NOT be replaced automatically as it could be from an attacker due to a compromised or stolen private key. Clients MAY provide a manual verification process that can be verified through a side-channel to be able to independently replace the setup event. The recovery setup event can assign anywhere from 1 to `n` backup keys assigned to be able to sign the change and revocation event. A threshold number of keys (`m` of `n`) can be assigned to verify this event.

```json
{
  "kind": x,
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

### Social Attestation Event

This is an event that is non-replaceable. Each client SHOULD provide a means to verify, sign and date recovery keys for those in their network for additional security. Clients can then make a public or private attestation that they have stored the recovery public keys for another's profile. In addition to clients storing the Backup Keys Setup Event for everyone in their social graph, they can also store these attestations on relays either public or private and encrypted. By making a public attestation, others in the network can see that you're able to verify the backup key for a profile; this can help built a robust fault-tolerant social graph.

```json
{
  "kind": x,
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

## Key Change and Revocation

### Event

This is a one-time event that is non-replaceable. Clients SHOULD implement a means to verify that users are aware of these conditions to avoid accidental broadcast of the event.

A client when receiving the event SHOULD present the user with a user interface to be able to verify the authenticity of the request, independently, and the provide a way to unfollow the identity and follow the new identity instead. A client MAY keep a record of the previous key for reference. Please see the external references section for additional guidance for UX design. The existence of a Backup Key Setup event is NOT REQUIRED to be able to handle a Key Change and Revocation Event, however it will not be able to use any backup recovery keys to verify and instead will only rely on the other two methods to verify.

### Verification for Clients

There are several ways to be able to verify the authenticity of a Key Change and Revocation Event. The primary one is through the use of the defined backup recovery keys, the second means is by trusting those whom have already verified the change that are in a social graph, the third way is through the use of external identities. All of these methods SHOULD be implemented, this specification only REQUIRES that at-least one of them is implemented. A client MUST NOT automatically, without user interaction, verify and switch to a new key and SHOULD present to the user a way to verify and accept the change.


### Verification for Relays

Relays MUST delete all events after receiving and verifying a Key Change and Revocation Event. For relays, the event MUST be verified by the primary key AND NOT the backup keys. A relay MUST not accept any further events from the key except additional key change and revocation events; this is to ensure that if a key is compromised and a fraudulent event is made, an honest event can also be made and broadcast. Each client can then verify which is honest.

## Acknowledgments

## External References
