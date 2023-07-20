NIP-XXX
------------------------------------
Git Remote Index and Commit Checkpoints
------------------------------------

`draft` `optional` `author:armstrys`

### Git Kinds
The goal of this nip is to introduce mechanisms for git remote repository discovery and commit checkpointing into the nostr protocol. Git is already decentralized by nature. Centralized clients like GitHub serve two primary purposes.
 - They establish one central remote repository as the source of truth.
 - They provide a platform for non-git metadata tracking including issues and comments.

A decentralized implementation on nostr should replace the central remote repository with discovery mechanisms that take advantage of the already decentralized nature of git and also provide nostr-native representations of commits that need to be referenced by external metadata not handled by git.

This nip introduces two new kinds to acheive this:
 - A **"git repository" parameterized replaceable event** (`kind:34617`) to provide updateable connection points to existing git servers that are easily discoverable and referenced by the second new kind...
 - A **"git checkpoint" event** (`kind:4617`) which has the sole purpose of providing a coordination reference point for key commits in a git repository and should follow a reply structure similar to the chain of commits in git.

With these two kinds it is possible to represent censoship resistant personal repositories, forked repositories, and even moderated repositories by integrating these kinds with [NIP-172](172.md).

> Note: This implementation assumes nothing about how the client will interact with git because it only aims to coordinate a layer above git to track metadata and discover existing git server locations. Clients would need to integrate authentication via other providers until git server implementations with nostr authentication are availble.

### Git Tags

Additionally, we introduce two application-specific tags that should be used in conjunction with the new git event kinds:
1. A `"c"` tag that takes a commit, a branch name, and a marker ('', `"head"`, `"compare"`, `"output"`) like so: `["c",  "<commit hash>", "<branch name>","<marker>"]`.
2. A `"git-history"` tag that provides a plain text history of commands that a user ran to generate a merge output (e.g. `“git checkout <base hash>\ngit merge <compare hash> —no-ff”`).
3. An `"auth-required"` tag that allows an author to publicize whether a remote requires authentication to access. Clients SHOULD assume that remotes with no `"auth-required"` default to `true` - the equivalent of `["auth-required", true]`

### Git Remote Definition

`Kind:34617` defines a replaceable event ([NIP-33](33.md)) that provides a url to a remote repository as it's `.content`. Using a replaceable event allows other events (like commit checkpoints) to reference this remote via `"a"` tag without concern that the links will break should the author need to change the location of the remote repository. The event SHOULD contain one or more `"c"` tags for any commit that define the `"head"` commit of each branch that the user author wants to make discoverable in the repository. The author can include other commits on each branch for key commits like releases.

```json
{
  "id": "<32-bytes lowercase hex-encoded SHA-256 of the the serialized event data>",
  "pubkey": "<32-bytes lowercase hex-encoded public key of the event creator>",
  "created_at": "<Unix timestamp in seconds>",
  "kind": 34617,
  "tags": [
    ["auth-required", false],
    ["a", "34550:<Community event author pubkey>:<d-identifier of the community>", "<Optional relay url>"],
    ["d", "<git remote name>"],
    ["c",  "<commit hash>", "<branch name>","head"],
    ["c",  "<commit hash>", "<branch name>",""],
    ["c",  "<commit hash>", "<other branch name>","head"],
  ], 
  "content": "<address to remote>"
}
```

### Git Checkpoint Definition

The usage of `"c"` marked tags help reference different git-related events, all using `kind:34617`:
1. An event with a two `"c"` tags marked `"head"` and `"output"` should be interpreted as a **standard commit checkpoint**.
2. An event with three `"c"` tags marked `"head"`, `"compare"`, and `"output"` should be interpreted as a **merge checkpoint**
3. An event with a `"c"` tag marked `"compare"`, but with no `"output"` commit should be interpreted as a pull/merge request
4. An event with at least one `"c"` tag but without a `"compare"` or `"output"` marker should be interpreted as a **release/tag** and should reply to the appropriate commit.
5. An bare `kind:34617` event with no `"c"` tag should be interpreted as a **comment** if it is a reply to another event or as an **issue** if it is not. 

A commit checkpoint SHOULD include at least one `"a"` tag to a `kind:34617` remote repository where the tagged git commit in question can be found. Clients may also query for matching `"c"` tags to discover other relevant remotes as needed.

Clients should interpret any `"a"` tag that includes `"34617:*"` as the first place to search for commits referenced in a `kind:4617` event.

Clients MUST use marked event tags to chain checkpoints by the commit history. Identifying the proper “root” and “reply” events allow other clients to follow and discover events in the same chain and forks. 

Both `kind:34617` and `kind:4617` events MAY include a [NIP-172](172.md) style `"a"` tag to establish a moderated repository. This may also help with repository remote discovery and organization as the `kind:34550` community event could suggest default remotes for the community.

An example of a standard commit checkpoint:

```json
{
  "id": "<32-bytes lowercase hex-encoded SHA-256 of the the serialized event data>",
  "pubkey": "<32-bytes lowercase hex-encoded public key of the event creator>",
  "created_at": "<Unix timestamp in seconds>",
  "kind": 4617,
  "tags": [
        ["c",  "<current head hash>", "<optional branch name>","head"],
        ["c", "<expected output hash>",  "<branch name>", "output"],
        ["a", "34617:<compare remote event author pubkey>:<compare remote name>"],
        ["e", "<event id of first checkpoint with output on checkpoint chain>", "<optional relay url>", "root id"],
        ["e", "<event id of previous checkpoint with output on checkpoint chain>", "<optional relay url>", "reply"],
    ["a", "34550:<Community event author pubkey>:<d-identifier of the community>", "<Optional relay url>"]
      ], 
  "content": "<description of commit>"
}
```

An example of a merge/pull request checkpoint:

```json
{
  "id": "<32-bytes lowercase hex-encoded SHA-256 of the the serialized event data>",
  "pubkey": "<32-bytes lowercase hex-encoded public key of the event creator>",
  "created_at": "<Unix timestamp in seconds>",
  "kind": 4617,
  "tags": [
        ["c", "<current head hash>", "<branch name>","head"],
        ["c", "<compare output hash>",  "<branch name>", "compare"],
        ["a", "34617:<compare remote event author pubkey>:<compare remote name>"],
        ["e", "<event id of first checkpoint with output on checkpoint chain>", "<optional relay url>", "root id"],
        ["e", "<event id of previous checkpoint with output on checkpoint chain>", "<optional relay url>", "reply"],
        ["a", "34550:<Community event author pubkey>:<d-identifier of the community>", "<Optional relay url>"]
    ], 
  "content": "<description of merge/pull request>"
}
```

An example of a merge checkpoint:

```json
{
  "id": "<32-bytes lowercase hex-encoded SHA-256 of the the serialized event data>",
  "pubkey": "<32-bytes lowercase hex-encoded public key of the event creator>",
  "created_at": "<Unix timestamp in seconds>",
  "kind": 4617,
  "tags": [
        ["git-history", "<plain text commands used to execute merge>"],
        ["c",  "<current head hash>", "<branch name>","head"],
        ["c", "<compare hash>",  "<optional branch name>", "compare"],
        ["c", "<expected output hash>",  "<branch name>", "output"],
        ["a", "34617:<compare remote event author pubkey>:<compare remote name>"],
        ["e", "<event id of first checkpoint with output on checkpoint chain>", "<optional relay url>", "root id"],
        ["e", "<event id of previous checkpoint with output on checkpoint chain>", "<optional relay url>", "reply"],
        ["e", "<event id of merge/pull request>", "<optional relay url>", "mention"],
        ["a", "34550:<Community event author pubkey>:<d-identifier of the community>", "<Optional relay url>"],
        ["p", "<hex pubkey of merge/pull request author>"]
      ], 
  "content": "<description of changes since last checkpoint>"
}
```
When establishing a merge checkpoint that references a merge/pull request the client should include the merge/pull request author as a `"p"` tag similar to the usage of replies in [NIP-10](10.md)
