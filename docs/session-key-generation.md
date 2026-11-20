# Generating session keys

`midnight-node generate-session-keys` puts a fresh, complete set of session keys into
the node's keystore and prints their public keys for the Cardano registration. The key
set is whatever the runtime defines, so a runtime upgrade that adds a key type needs no
tool change.

## Running it

Run it on the validator's machine, as the user the node runs as, so the new key files
get the node's owner and permissions. The node can keep running: it sees the new keys
without a restart. In Docker:

```sh
docker exec -u <node-user> <container> \
  midnight-node generate-session-keys --keystore-path <node keystore directory> \
  --rpc-port <node RPC port>
```

`--keystore-path` is required: give the directory the running node uses as its
keystore, by default `<base-path>/chains/<chain id>/keystore`. If the keystore has a
password, pass it with `--password-filename` or `--password-interactive` (`--password`
would leave it in the process list and shell history). The tool reads no node
configuration.
Before writing, it prints the keystore's absolute path, with symbolic links resolved,
and its `crch` key to stderr, so you can check it is the node's keystore.

`--runtime-wasm <path>` generates the keys that runtime file defines, compressed
(`.compact.compressed.wasm`) or not, for example an upgrade that adds a key type and is
not on the chain yet. The tool then reads nothing from the chain and does not check the
file against it: make sure it is the runtime the network will run.

Without `--runtime-wasm` the node must be running: the tool reads the chain's current
runtime code over the node's RPC on `127.0.0.1`, at the port given with `--rpc-port`
(required in this case, there is no default, and rejected together with
`--runtime-wasm`).

## What it checks

Before writing anything, the tool aborts if (the keystore is checked before the runtime
code is read):

- the keystore directory does not exist (it is never created);
- the keystore does not hold exactly one `crch` (cross-chain) key;
- the keystore cannot sign with that `crch` key, which means a wrong keystore password.
  A `crch` key stored as a hex seed (as `CROSS_CHAIN_SEED_FILE` inserts it) signs with any
  password, so a wrong password goes undetected there;
- the runtime does not run with this node's host functions;
- the runtime does not have `SessionKeysGenerationApi` version 1, as no runtime before
  this tool does.

The `crch` key is never rotated. Existing key files are never opened, except for the
`crch` key, which the keystore uses for that one test signature; the tool never sees its
secret and lists every other public key by file name only.

If something fails after keys were written, nothing is deleted and the error names
the new keystore files. This includes a runtime that writes a key it does not return as
a session key.

## Output

```json
{"sidechain_pub_key":"0x..","keys":{"aura":"0x..","gran":"0x..","babe":"0x..","beef":"0x.."},"signatures":{"aura":"0x..","gran":"0x..","babe":"0x..","beef":"0x.."},"runtime_code_hash":"0x.."}
```

`sidechain_pub_key` is the `crch` public key that identifies the registration.
`keys` and `signatures` list the keys in the runtime's key order. `signatures` holds each
key's proof of possession, as the runtime makes it: its signature over `b"POP_"` followed
by the SCALE-encoded `AccountId32` of the `crch` key, the statement `pallet_session` uses for
an ownership proof. Joined in that order they form the `pallet_session` ownership proof for
that account.
`runtime_code_hash` is the Blake2-256 hash of the runtime code the keys came from.

## After generating

Register the new public keys on Cardano. Keep the old keys in the keystore until the
new registration is active, two Cardano epochs later.

Keys from `*_SEED_FILE` files (`AURA_SEED_FILE`, `BABE_SEED_FILE`, `GRANDPA_SEED_FILE`,
`BEEFY_SEED_FILE`) are re-inserted on every start, so after a rotation those files
must be updated or removed.

## AURA to BABE migration

Generate the keys with `--runtime-wasm` and the runtime that adds BABE: the chain's
runtime before the upgrade has no `SessionKeysGenerationApi` and does not define the
`babe` key.

The tool generates a complete new key set, with its proofs of possession for later use.
For the AURA to BABE migration, submit only the new `babe` and `beef` public keys for
the Cardano registration. Your existing keys stay in the keystore, unchanged, and the
node keeps producing blocks with them; the other new keys simply stay unused.

## Startup check

On startup a validator looks up its own entry in the permissioned and registered
candidate lists on Cardano by its `crch` key, and logs (target `session-keys`) for
each registered session key whether it is `present in keystore` or `MISSING from
keystore`. Without a `crch` key it logs that and keeps running. The check runs once:
if the Cardano data is not available yet, it logs a warning and does not retry.
