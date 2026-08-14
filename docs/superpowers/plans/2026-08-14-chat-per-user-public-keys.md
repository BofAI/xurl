# XChat Per-User Public-Key Retrieval Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `xurl chat read` obtain participant signing keys through `GET /2/users/{id}/public_keys` instead of the enrollment-sensitive batch endpoint.

**Architecture:** Keep the public `GetChatUsersPublicKeys` interface and every CLI/Chat XDK caller unchanged. Implement that helper by composing the existing single-user `GetChatPublicKeys` function, tagging each returned key with its owner, and failing atomically with user-specific context.

**Tech Stack:** Go 1.24.2, `net/http/httptest`, Testify, official `github.com/xdevplatform/chat-xdk/go/chatxdk` dependency.

## Global Constraints

- Do not change OAuth behavior, local private-key storage, read receipts, or Chat XDK encryption/decryption.
- Do not add custom cryptography or runtime dependencies.
- Return no partial key set when any participant lookup fails.
- Preserve the upstream MIT license.

---

### Task 1: Retrieve and tag participant keys per user

**Files:**
- Modify: `api/chat.go:93-120`
- Test: `api/chat_test.go`

**Interfaces:**
- Consumes: `GetChatPublicKeys(client Client, userID string, opts RequestOptions) ([]ChatPublicKey, error)`.
- Produces: unchanged `GetChatUsersPublicKeys(client Client, userIDs []string, opts RequestOptions) ([]ChatPublicKey, error)`; every returned row has `UserID` set to its requested owner.

- [ ] **Step 1: Write the failing route-and-owner regression test**

Add this test after `TestGetChatPublicKeys` in `api/chat_test.go`:

```go
func TestGetChatUsersPublicKeysUsesPerUserRoutes(t *testing.T) {
	var paths []string
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		paths = append(paths, r.URL.Path)
		w.Header().Set("Content-Type", "application/json")
		switch r.URL.Path {
		case "/2/users/7/public_keys":
			_, _ = w.Write([]byte(`{"data":[{"public_key_version":"1700","public_key":"idpk7","signing_public_key":"sigpk7","identity_public_key_signature":"binding7"}]}`))
		case "/2/users/8/public_keys":
			_, _ = w.Write([]byte(`{"data":[{"public_key_version":"1800","public_key":"idpk8","signing_public_key":"sigpk8","identity_public_key_signature":"binding8"}]}`))
		default:
			http.Error(w, `{"error":"unexpected route"}`, http.StatusNotFound)
		}
	}))
	defer server.Close()
	client := chatTestClient(t, server)

	keys, err := GetChatUsersPublicKeys(client, []string{"7", "8"}, RequestOptions{})
	require.NoError(t, err)
	require.Len(t, keys, 2)
	assert.Equal(t, []string{"/2/users/7/public_keys", "/2/users/8/public_keys"}, paths)
	assert.Equal(t, "7", keys[0].UserID)
	assert.Equal(t, "1700", keys[0].Version)
	assert.Equal(t, "8", keys[1].UserID)
	assert.Equal(t, "1800", keys[1].Version)
}
```

- [ ] **Step 2: Run the focused test and verify RED**

Run: `go test ./api -run '^TestGetChatUsersPublicKeysUsesPerUserRoutes$' -count=1 -v`

Expected: FAIL because the current implementation requests `/2/users/public_keys?ids=7%2C8`, which the controlled server rejects.

- [ ] **Step 3: Implement per-user retrieval**

Replace `GetChatUsersPublicKeys` in `api/chat.go` with:

```go
// GetChatUsersPublicKeys fetches registered public keys for the given users;
// each returned row carries its owner's user_id. Public keys are fetched via
// the per-user route because the batch route is not enabled for every X app.
func GetChatUsersPublicKeys(client Client, userIDs []string, opts RequestOptions) ([]ChatPublicKey, error) {
	if len(userIDs) == 0 {
		return nil, nil
	}
	var keys []ChatPublicKey
	for _, userID := range userIDs {
		userKeys, err := GetChatPublicKeys(client, userID, opts)
		if err != nil {
			return nil, err
		}
		for i := range userKeys {
			userKeys[i].UserID = userID
		}
		keys = append(keys, userKeys...)
	}
	return keys, nil
}
```

- [ ] **Step 4: Run the focused test and verify GREEN**

Run: `go test ./api -run '^TestGetChatUsersPublicKeysUsesPerUserRoutes$' -count=1 -v`

Expected: PASS with two per-user requests and correctly tagged result rows.

- [ ] **Step 5: Commit the first behavior**

```bash
git add api/chat.go api/chat_test.go
git commit -m "fix(chat): fetch participant keys per user"
```

### Task 2: Preserve atomic failure with user context

**Files:**
- Modify: `api/chat.go:96-116`
- Test: `api/chat_test.go`

**Interfaces:**
- Consumes: the per-user implementation from Task 1.
- Produces: an error containing `user <id>` and a nil result when any requested user's lookup fails.

- [ ] **Step 1: Write the failing atomic-error regression test**

Add this test after `TestGetChatUsersPublicKeysUsesPerUserRoutes`:

```go
func TestGetChatUsersPublicKeysIdentifiesFailedUserAndReturnsNoPartialKeys(t *testing.T) {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		if r.URL.Path == "/2/users/7/public_keys" {
			_, _ = w.Write([]byte(`{"data":[{"public_key_version":"1700","public_key":"idpk7","signing_public_key":"sigpk7","identity_public_key_signature":"binding7"}]}`))
			return
		}
		w.WriteHeader(http.StatusForbidden)
		_, _ = w.Write([]byte(`{"error":"denied"}`))
	}))
	defer server.Close()
	client := chatTestClient(t, server)

	keys, err := GetChatUsersPublicKeys(client, []string{"7", "8"}, RequestOptions{})
	require.Error(t, err)
	assert.Nil(t, keys)
	assert.Contains(t, err.Error(), "user 8")
}
```

- [ ] **Step 2: Run the focused test and verify RED**

Run: `go test ./api -run '^TestGetChatUsersPublicKeysIdentifiesFailedUserAndReturnsNoPartialKeys$' -count=1 -v`

Expected: FAIL because Task 1 returns the API error without identifying user `8`.

- [ ] **Step 3: Add user-specific error context**

Change the error branch in `GetChatUsersPublicKeys` to:

```go
		if err != nil {
			return nil, fmt.Errorf("failed to fetch public keys for user %s: %w", userID, err)
		}
```

- [ ] **Step 4: Run focused and complete verification**

Run:

```bash
go test ./api -run '^TestGetChatUsersPublicKeys' -count=1 -v
go test ./... -count=1
mkdir -p bin
CGO_ENABLED=1 go build -o ./bin/xurl-digest .
./bin/xurl-digest version
```

Expected: both focused tests and the full suite pass; the CGO-enabled fork binary builds and prints its version.

- [ ] **Step 5: Commit the completed behavior**

```bash
git add api/chat.go api/chat_test.go
git commit -m "fix(chat): identify participant key lookup failures"
```
