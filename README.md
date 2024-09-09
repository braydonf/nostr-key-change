NIP-X
=====

Key Migration and Revocation
------

`draft` `optional`

This NIP defines a protocol for clients and relays to gracefully recovery from a compromised private key.

At a minimum this includes the revocation of a private key. Clients give warning that the key is compromised. Relays and clients reject future events from a revoked key and may delete existing events.

Also defined is a protocol for a user to be able to remember and associated a public key with a user such that if a key is compromised, the user is able to identity who it was and how to go about recovering from the compromise. This includes an ability to remember the original name, NIP-05 identification, websites, recovery keys and other associated metadata. Client implementations can provide various strategies to help that recovery, starting a simple as displaying that the key has been compromised.

There are two new events introduced:

* [Key Migration and Revocation](#key-migration-and-revocation-event)
* [User Metadata Attestation](#user-metadata-attestation-event)

There is one new field introduced for a user's metadata (`kind 0`) of `recovery_pubkeys`:
* [Recovery Keys](#recovery-keys)

## Key Migration and Revocation Event

This is a regular event with kind `50`. It will revoke a public key (it has been compromised) and stop future events from the key. It will also provide a means to inform followers of the compromise and suggest a new public key to follow.

```js
{
  "kind": 50,
  "pubkey": "<user-pubkey>",
  "tags": [
	["new-key", "<new-pubkey>"],
	["key-migration"],
	["recovery-sigs", "<index-0-sig", "<index-1-sig>", "<index-2-sig>"]
  ],
  "content": "<optional-comment>"
}
```

* If a `new-key` IS provided:
  * The `key-migration` tag MUST be included, once and without a value.
  * There MUST NOT be multiple `new-key` tags or mulitple values.
  * The `recovery-sigs` tag MAY be included with the signatures for `m` of `n` recovery public keys.
* If a `new-key` IS NOT provided:
  * The `key-revocation` tag MUST be included, once and without a value.

### Event Handling for Clients

For a client, this event is both a revocation with a suggestion for migration. The revocation MUST be handled by verifying only the event signature and MUST be automatic. The migration, if it is provided, MUST NOT be automatic and MAY be presented and verified by the user to accept or reject the migration key change.

#### Key Revocation
* Upon a valid key revocation:
  * All events SHOULD display a warning that the event is from a revoked public key.
  * A user's profile SHOULD display a warning that the key has been compromised.
  * All events of a revoked public key MAY be deleted, this MAY be after a duration of time depending on the client or preferences.

#### Key Migration
* If a user has made a prior _User Metadata Attestation_:
  * The user interface MAY display the original name, NIP-05, recovery keys (with signature verification) and other attested to user metadata.
  * The user interface MAY provide a means to accept or reject a suggested key migration. This can include verifying using NIP-05, recovery keys, a social graph of those the user follows that are now following the suggested new key.
* Upon the user accepting a suggested new key:
  * The _old key_ SHOULD be unfollowed and the _new key_ SHOULD be followed.
  * The _old key_ SHOULD be added to a mute list.

### Event Handling for Relays

For a relay, this event is a key revocation.

#### Key Revocation
* Upon a valid key revocation:
  * All future events, as determined when it was received and not the date on the event, of a revoked public key MUST be rejected except for another _Key Migration and Revocation Event_. This is to ensure that if a key is compromised and a fraudulent event is made, an honest event can also be made and broadcast.
  * All events of a revoked public key MAY be deleted. The time-frame that events are deleted MAY be defined by an agreed upon terms between client and relay.
* For denial-of-service mitigation, a relay MAY require proof-of-work, a small fee or another solution to continue to write _Key Migration and Revocation Events_. This SHOULD be determined by the terms agreed upon by the client and relay.

## User Metadata Attestation Event

This is a parameterized replaceable event with kind `30051`. This should be an attestation for another user's metadata `kind 0`. This will help a user remember what public key is associated with what `display_name`, `nip05`, `website` and other metadata, should that `kind 0` event be compromised in the future. It can also attest to a newly defined `recovery_pubkeys` field that could later be useful to be able to verify a user should their private key be compromised. The attestation can include both _public_ and _private_ information.

Public:
```js
{
  "kind": 30051,
  "pubkey": "<user-pubkey>",
  "tags": [
	["d", "<pubkey-of-friend>"],
	["p", "<pubkey-of-friend>"],
	["p", "<optional-predecessor-pubkey-of-friend>"],
	["attestations", JSONStringify({"<metadata-key>": "<metadata_value>"})]
  ],
  "content": ""
}
```

Private:
```js
  "kind": 30051,
  "pubkey": "<user-pubkey>",
  "tags": [
	["d", "<encrypted-and-hashed-pubkey"]
  ],
  "content": Nip44Encrypt(JSONStringify([
	["p", "<pubkey-of-friend>"],
	["attestations", JSONStringify(["<metadata-key>", "<metadata-key-2>"])]
  ]))
```

* For a _public_ attestation:
  * The `d` tag and a `p` tag MUST include a public key for the attested to metadata.
  * Another `p` tag SHOULD be included if there was a predecessor public key. This helps to inform other users of a link between the predecessor public and and a new public key.
  * The `attestations` tag MUST include JSON stringified copy of the attested to metadata keys.
* For a _private_ attestation:
  * The `d` tag MUST be an encrypted and hashed version of the public key, and MUST be the hex encoding of a sha256 hash of an encrypted, with NIP-44, of the public key.
  * The `p`, `metadata` and `attestations` tags, as the same as the public attestation, MUST be JSON stringified and NIP-44 encrypted in the content field.

## Recovery Keys

This is a new field on a `kind 0` event with a key of `recovery_keys`. Its purpose is to define a set of recovery keys that can be used to help migrate to a new key in the future, if it becomes necessary. The event can assign anywhere from `1` to `n` recovery keys assigned to be able to sign a _Key Migration and Revocation Event_. A threshold number of keys (`m` of `n`) can be assigned to verify this event.

The value should be as follows:

```js
[<threshold>, <recovery-pubkey-1>, <recovery-pubkey-2>]
```

* Clients MAY present a user interface to make an attestation, for future reference, if this field is available an the metadata.
* Clients MAY use hardware devices and NIP-06 seed phrases to backup the recovery keys.
