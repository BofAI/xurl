# XChat Per-User Public-Key Retrieval Design

## Goal

Make `xurl chat read` decrypt conversations for X apps where the batch participant-key endpoint returns `client-not-enrolled`, while keeping the official Chat XDK and all existing encrypted-event handling unchanged.

## Selected approach

Change `api.GetChatUsersPublicKeys` to retrieve each requested user's keys through the existing `GET /2/users/{id}/public_keys` wrapper. Add the requested user ID to each returned key before merging the results. Callers continue to receive the same `[]ChatPublicKey` shape, so the CLI decryption path does not change.

This deliberately trades one batch request for one request per unique participant. Digest conversations are small, and using the endpoint already confirmed to work is more predictable than adding error-string-based fallback behavior.

## Alternatives considered

- Batch first, then fall back only on `client-not-enrolled`: fewer requests when the batch endpoint works, but couples the client to an unstable platform error representation and still makes the failing request first.
- Keep the external JavaScript adapter: avoids a fork, but leaves two execution paths and prevents `xurl chat read` from working as the single command.

## Scope

- Modify only the participant signing-key retrieval implementation and its API tests.
- Preserve public function signatures, OAuth behavior, local key storage, read-receipt behavior, and Chat XDK encryption/decryption.
- Preserve the upstream MIT license and use a separate fork branch.
- Do not add custom cryptography or new runtime dependencies.

## Error handling

Return the first per-user request error with the affected user ID in the message. Do not return a partial key set, because partial signing keys can make valid events appear unverifiable. Empty input continues to return an empty result without making requests.

## Verification

- Add a regression test proving multiple user IDs cause per-user endpoint requests and that every returned key is tagged with its owner.
- Add a regression test proving a per-user failure identifies the failed user and returns no partial result.
- Run `go test ./...`.
- Build the Chat-enabled binary with `CGO_ENABLED=1 go build ./...` on the current supported macOS environment.
