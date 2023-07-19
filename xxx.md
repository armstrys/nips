NIP-XXX
------------------------------------
Git Remote Index and Commit Checkpoints
------------------------------------

`draft` `optional` `author:armstrys`

# Git Kinds
The goal of this nip is to introduce mechanisms for git remote repository discovery and commit checkpointing into the nostr protocol. Git is already decentralized by nature. Centralized clients like GitHub serve two primary purposes. First, they provide one central remote repository. Secondly, they provide a platform for non-git metadata tracking. 

A decentralized implementation on nostr should replace the central remote repository with discovery mechanisms that take advantage of the already decentralized nature of git. To do so, this nip introduces a git repository kind and a git checkpoint kind that establishes a nostr event for key commits needed for metadata tracking via nostr. 

This nip introduces two new kinds:
    - a git repository parameterized replaceable kind (34617) to provide updateable connection points to existing git repositories or servers
    - and a git checkpoint event kind (4617) which has the sole purpose of providing a coordination reference point for key commits in a git repository.

# Git Tags

Additionally, we introduce two application-specific tags that should be used in conjunction with the git checkpoint kind
1. A ”c” tag that takes a commit, an optional branch name, and a marker ('', 'head',  'compare', 'output') like so `["c",  "<commit hash>", "<branch name>","<marker>"]`
2. A "git-history" tag that provides a plain text history of commands that a user ran to generate a merge output (e.g. `“git checkout <base hash>\ngit merge <compare hash> —no-ff”`).

# Git Remote Definition

Using the git-branch tag, a repository location can be defined as:

```json
{
  "id": "<32-bytes lowercase hex-encoded SHA-256 of the the serialized event data>",
  "pubkey": "<32-bytes lowercase hex-encoded public key of the event creator>",
  "created_at": "<Unix timestamp in seconds>",
  "kind": "34617",
  "tags": [
    ["a", "34550:<Community event author pubkey>:<d-identifier of the community>", "<Optional relay url>"],
    ["d", "<git remote name>"],
	  ["c",  "<commit hash>", "<branch name>","head"],
	  ["c",  "<commit hash>", "<branch name>",""],
	  ["c",  "<commit hash>", "<other branch name>","head"],
  ], 
  "content": "<address to remote>"
}
```

# Git Checkpoint Definition

The usage of “c” marked tags should can differentiate between 5 usages of kind 34617:
1. An event with a “c” tag and no marker or “” should be interpreted as a release/tag and should reply to the appropriate commit. Any other commit tags should be ignored
2. An event with “c” tags marked “compare” and “output” should be interpreted as a merge
3. An event with a “c” tag of “output”, but not “compare” should be interpreted as a new commit with no merge
4. An event with a “c” tag marked “compare”, but with no “output” commit should be interpreted as a pull/merge request
5. An event with no “c" tag should be interpreted as a comment if it is a reply to another event or as an issue if it is not. 

Clients MUST use marked event tags to chain checkpoints by the commit history. Identifying the proper “root” and “reply” events allow other clients to follow and discover events in the same chain and forks. 

Because both git checkpoint events and git remote events contain branch tags

Git checkpoints may be used within the context of NIP-172 to introduce a moderated repository of git checkpoints. 

Examples:
A basic commit can look like

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
	  ["e", "<event id of merge/pull request>", "<optional relay url>", "mention"],
    ["a", "34550:<Community event author pubkey>:<d-identifier of the community>", "<Optional relay url>"]
      ], 
  "content": "<description of commit>"
}
```

A merge/pull request can look like

```json
{
  "id": "<32-bytes lowercase hex-encoded SHA-256 of the the serialized event data>",
  "pubkey": "<32-bytes lowercase hex-encoded public key of the event creator>",
  "created_at": "<Unix timestamp in seconds>",
  "kind": 4617,
  "tags": [
	  ["c",  "<current head hash>", "<branch name>","head"],
	  ["c", "<compare output hash>",  "<branch name>", "compare"],
	  ["a", "34617:<compare remote event author pubkey>:<compare remote name>"],
	  ["e", "<event id of first checkpoint with output on checkpoint chain>", "<optional relay url>", "root id"],
	  ["e", "<event id of previous checkpoint with output on checkpoint chain>", "<optional relay url>", "reply"],
	  ["e", "<event id of merge/pull request>", "<optional relay url>", "mention"],
    ["a", "34550:<Community event author pubkey>:<d-identifier of the community>", "<Optional relay url>"]
      ], 
  "content": "<description of merge/pull request>"
}
```

A merge checkpoint can be defined as a reply chain on the existing checkpoints within the same branch.

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
    ["a", "34550:<Community event author pubkey>:<d-identifier of the community>", "<Optional relay url>"]
      ], 
  "content": "<description of changes since last checkpoint>"
}
```

Merge replies should include an "author" tag as an extension of  nip-10 that can be used to designate authors involved in the checkpoint

Clients should interpret any “a” tag that includes “34617:*” as the first place to search for commits referenced in a kind 4617 event.

Further remote discovery is available by filtering “git-branch” tag matches between kind 4617 and 34617 events. This discovery is simplified further by the use of moderated communities and the linked reply structure of checkpoint commits. With moderated communities, moderators can also designate specific repositories by approving them. 
