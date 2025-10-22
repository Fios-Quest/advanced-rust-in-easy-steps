Typestate
=========

The typestate pattern encodes state into type information. This is useful because this information exists when you
write the program but not when you run it, allowing you to enforce domain logic with minimal overhead.

State
-----

Lets imagine a simplified pull request process. We can open a new pull request, which then must be approved by at least
one person, before it's merged. Multiple people can approve it, but if anyone rejects it (even after its approved) it
can no longer be merged.

```mermaid
%%{ init: { 'flowchart': {'defaultRenderer': 'elk' } } }%%
flowchart LR
    A --->|foo| B
    B --->|bar| B
````
```mermaid
---
config:
  layout: elk
---
flowchart LR
    O[Open]
    A[Approved]
    R[Rejected]
    M[Merged]

    O -- Approve --> A
    A -- Approve --> A
    O -- Reject --> R
    A -- Reject --> R
    A -- Merge --> M
```

Let's try to model one small part of this flow in a traditional way, the "approve" action.

First lets model it with a PullRequest type that _has_ a status. A PR can be approved if the status is ReadyForReview or
Approved. If the status was not one of those, then we'll need to return an error to say that the status could not be
changed.

```rust
struct InvalidStateError;

#[derive(Debug, PartialEq)]
enum Status {
    Open,
    Approved,
    Rejected,
    Merged
}

struct PullRequest {
    status: Status,
    // ...other PR details ...
}

impl PullRequest {
    fn approve(&mut self) -> Result<(), InvalidStateError> {
        if self.status == Status::Open || self.status == Status::Approved {
            self.status = Status::Approved;
            Ok(())
        } else {
            Err(InvalidStateError)
        }
    }

    fn reject(&mut self) -> Result<(), InvalidStateError> {
        if self.status == Status::Open || self.status == Status::Approved {
            self.status = Status::Rejected;
            Ok(())
        } else {
            Err(InvalidStateError)
        }
    }

    fn merge(&mut self) -> Result<(), InvalidStateError> {
        if self.status == Status::Approved {
            self.status = Status::Merged;
            Ok(())
        } else {
            Err(InvalidStateError)
        }
    }
}
# 
# fn main () {
// You can approve a PR that's just been opened or already Approved
let mut pr = PullRequest { status: Status::Open };
assert!(pr.approve().is_ok());
assert_eq!(pr.status, Status::Approved);
assert!(pr.approve().is_ok());

// You can not approve or merge rejected PR
let mut pr = PullRequest { status: Status::Rejected };
assert!(pr.approve().is_err());
assert!(pr.merge().is_err());
# }
```

We've written up our three state change operations but we've had to write a lot of branching logic into each method to
check that the operation is only being run on a PR in a valid state. If it wasn't in a valid state we produce errors
that now need to be dealt with.


```rust
struct PullRequestOpen {
    // ...other PR details ...
}

struct PullRequestApproved {
    // ...other PR details ...
}

struct PullRequestRejected {
    // ...other PR details ...
}

struct PullRequestMerged {
    // ...other PR details...
}

impl PullRequestOpen {
    fn approve(self) -> PullRequestApproved {
        PullRequestApproved {
            // ... move self into PullRequestApproved ...
        }
    }

    fn reject(self) -> PullRequestRejected {
        PullRequestRejected {
            // ... move self into PullRequestApproved ...
        }
    }
}

impl PullRequestApproved {
    fn approve(self) -> PullRequestApproved {
        self
    }

    fn reject(self) -> PullRequestRejected {
        PullRequestRejected {
            // ... move self into PullRequestApproved ...
        }
    }

    fn merge(self) -> PullRequestMerged {
        PullRequestMerged {
            // ... move self into PullRequestApproved ...
        }
    }
}

# 
# fn main () {
let open_pr = PullRequestOpen { };

// You can not merge an open PR, the next line won't compile
// let merged_pr = open_pr.merge();

// You can approve a PR with Ready for Review or already Approved
let approved_pr = open_pr.approve();
let still_approved = approved_pr.approve();

// Then it can be merged
let merged_pr = still_approved.merge();

// The `.approve()` method doesn't exist for rejected PRs, commented line won't compile
let rejected_pr = PullRequestRejected { };
// rejected_pr.approve();
# }
```

Pros and Cons
-------------

Utilising the typestate pattern requires writing a lot more code _but_ that code has fewer branches, fewer (or in this 
case no) potential error states, and everything is logically subdivided making maintaining the code more trivial.

In the example we've given here, we consume the type each time we change state. Generally in Rust it's better to pass
references rather than ownership, but in the case of transitioning state we want to return a new type. Often the old
type will contain owned data that needs to be moved to the new type, and we'll rarely want to keep the old state around
but your needs may differ. If you regularly need to keep the old state you could pass by reference, but if you only
occasionally need to keep the old state, you could clone it first.

