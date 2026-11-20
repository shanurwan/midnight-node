#runtime #session-keys

# Add `SessionKeysGenerationApi`

`SessionKeysGenerationApi_generate_session_keys_with_proofs(cross_chain)` generates a fresh
set of the runtime's session keys into the keystore and returns, in key order, each key's
type, public key and its own proof of possession for the candidate's `crch` key. The
runtime defines what the proof signs; currently it is the statement `pallet_session` uses
for the account of the `crch` key. `midnight-node generate-session-keys` requires this API.

PR: https://github.com/midnightntwrk/midnight-node/pull/2488
