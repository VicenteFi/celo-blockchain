# Specification for Celo IBFT protocol

## Celo IBFT specification

### High level overview

The Celo IBFT protocol is a BFT (Byzantine Fault Tolerant) protocol that allows
a group of participants to agree on the ordering of values by exchanging voting
messages across a network. As long as less than 1/3rd of participants deviate
from the protocol then the protocol should ensure that there is only one
ordering of values that participants agree on and also that and that they can
continue to agree on new values, i.e. they don't get stuck.

Ensuring that only one ordering is agreed upon is referred to as 'safety' and
ensuring that the participants can continue to agree on new values is referred
to as 'liveness'.

### System model

We consider a group of `3f+1` participating computers (participants) that are
able to communicate across a network. The group should be able to tolerate `f`
failures without losing liveness or safety.

All participants know the other participants in the system and it is assumed
that all messages are signed in some way such that participants in the protocol
know which participant a message came from and that messages cannot be forged.

Only messages from participants are considered.

For messages containing a value only messages with valid values are considered.

Participants are able to broadcast messages to all other participants. In the
case that a participant is off-line or somehow inaccessible they will not
receive broadcast messages and there is no general mechanism for these messages
to be re-sent.

We refer to a consensus instance to mean consensus for a specific height,
consensus instances are independent (they have no shared state).

#### Instance state
These are the variables that are held by each instance.

`CURRENT_STATE`\
`CURRENT_HEIGHT`\
`CURRENT_ROUND`\
`DESIRED_ROUND`\
`CURRENT_VALUE`\
`CURRENT_PREPARED_CERTIFICATE`


### Algorithm
See supporting [functions](#Functions) and [notation](#Appendix-1-Notation).
```
upon: FinalCommittedEvent
  CURRENT_HEIGHT ← CURRENT_HEIGHT+1
  CURRENT_ROUND ← 0
  DESIRED_ROUND ← 0
  CURRENT_STATE ← AcceptRequest
  CURRENT_VALUE ← nil
  schedule onRoundChangeTimeout(CURRENT_HEIGHT, 0) after roundChangeTimeout(0)

upon: <RequestEvent, CURRENT_HEIGHT, VALUE> && CURRENT_STATE = AcceptRequest
  if CURRENT_ROUND = 0 && isProposer(CURRENT_HEIGHT, DESIRED_ROUND) {
    bc(<Preprepare, CURRENT_HEIGHT, 0, VALUE, nil>)
  }

upon: <Preprepare, CURRENT_HEIGHT, DESIRED_ROUND, VALUE, ROUND_CHANGE_CERTIFICATE> from proposer(CURRENT_HEIGHT, DESIRED_ROUND) && CURRENT_STATE = AcceptRequest
  if (DESIRED_ROUND > 0 && validRCC(CURRENT_HEIGHT, DESIRED_ROUND, VALUE, ROUND_CHANGE_CERTIFICATE)) || (DESIRED_ROUND = 0 && ROUND_CHANGE_CERTIFICATE = nil)  {
    CURRENT_ROUND ← DESIRED_ROUND
    CURRENT_VALUE ← V
    CURRENT_STATE ← Preprepared
    bc(<Prepare, CURRENT_HEIGHT, DESIRED_ROUND, CURRENT_VALUE>)
  }

upon: M ← { <T, CURRENT_HEIGHT, DESIRED_ROUND, CURRENT_VALUE> : T ∈ {Prepare, Commit} } && |M| > 2f+1 && CURRENT_STATE ∈ {AcceptRequest, Preprepared} 
  CURRENT_STATE ← Prepared
  CURRENT_PREPARED_CERTIFICATE ← <PreparedCertificate, M, CURRENT_VALUE>
  bc(<Commit, CURRENT_HEIGHT, DESIRED_ROUND, CURRENT_VALUE>)

upon: M ← { <Commit, CURRENT_HEIGHT, DESIRED_ROUND, CURRENT_VALUE> } && |M| > 2f+1 && CURRENT_STATE ∈ {AcceptRequest, Preprepared, Prepared} 
  CURRENT_STATE ← Committed
  deliverValue(CURRENT_VALUE)

upon: m<RoundChange, CURRENT_HEIGHT , ROUND, PREPARED_CERTIFICATE> && (PREPARED_CERTIFICATE = nil || validPC(PREPARED_CERTIFICATE)) 
  if R < DESIRED_ROUND {
    send(<RoundChange, CURRENT_HEIGHT, DESIRED_ROUND, CURRENT_PREPARED_CERTIFICATE>, sender(m))
  } else if quorumRound() > DESIRED_ROUND {
	DESIRED_ROUND ← quorumRound()
	CURRENT_ROUND ← quorumRound()
    schedule onRoundChangeTimeout(CURRENT_HEIGHT, DESIRED_ROUND) after roundChangeTimeout(DESIRED_ROUND)
	if CURRENT_VALUE != nil && isProposer(CURRENT_HEIGHT, CURRENT_ROUND) {
      bc(<RoundChange, CURRENT_HEIGHT, CURRENT_ROUND, CURRENT_VALUE, CURRENT_PREPARED_CERTIFICATE>)
	}
  } else if f1Round() > DESIRED_ROUND {
    DESIRED_ROUND ← f1Round() 
    CURRENT_STATE ← WaitingForNewRound
    schedule onRoundChangeTimeout(CURRENT_HEIGHT, DESIRED_ROUND) after roundChangeTimeout(DESIRED_ROUND)
    bc(<RoundChange, CURRENT_HEIGHT, DESIRED_ROUND, CURRENT_PREPARED_CERTIFICATE>)
  }
```

### Functions

#### Application provided functions
No pseudocode is provided for these functions since their implementation is
application specific.

`proposer(HEIGHT, ROUND)`\
Returns the proposer for the given height and round.

`isProposer(HEIGHT, ROUND)`\
Returns true if the current participant is the proposer for the given height
and round. 

`deliverValue(VALUE)`\
Delivers the given value to the application.

`roundChangeTimeout(ROUND)`\
Returns the timeout for the given round 

`bc(<Preprepare, HEIGHT, ROUND, VALUE>)`\
Broadcasts the given message to all connected participants. 

`send(<Commit, HEIGHT, ROUND, VALUE>, sender(m))`\
Sends the given message to to the sender of another message.

#### PCRound
Asserts that all messages in the given prepared certificate share the same round and returns that round.
```
PCRound(<PreparedCertificate, M, *>) {
  ∃ R : ∀ m<*, *, ROUNDm, *> ∈ M : R = ROUNDm
  return R
}
```

#### PCValue
Return the value associated with a prepared certificate.
```
PCValue(<PreparedCertificate, *, VALUE>) {
  return VALUE
}
```

#### validPC

Returns true if the message set contains prepare or commit messages from at
least 2f+1 and no more than 3f+1 participants for current height, matching the
prepared certificate value and all sharing the same height and round.
```
validPC(<PreparedCertificate, M, VALUE>) {
  N ← { m<MESSAGE_TYPE, CURRENT_HEIGHT, *, VALUEm> ∈ M : (MESSAGE_TYPE = Prepare || MESSAGE_TYPE = Commit) && VALUEm = VALUE } 
  return 2f+1 <= |N| <= 3f+1 &&
  ∀ m<*, HEIGHTm, ROUNDm, *>, n<*, HEIGHTn, ROUNDn, *> ∈ N : HEIGHTm = HEIGHTn && ROUNDm = ROUNDn 
}
```

#### validRCC

Returns true if the round change contains at least 2f+1 and no more than 3f+1
round changes that match the given height and have a round greater or equal
than the given round and either have a valid prpared certificate or no prpared
certificate. If any round change certificates have a prepared certificate,
then there must exist one with greater than or equal round to all the others
and with a value of V.

```
validRCC(HEIGHT, ROUND, VALUE, ROUND_CHANGE_CERTIFICATE) {
  M ← { m<RoundChange, HEIGHTm , ROUNDm, PREPARED_CERTIFICATE> ∈ ROUND_CHANGE_CERTIFICATE : HEIGHTm = HEIGHT && ROUNDm >= ROUND && (PREPARED_CERTIFICATE = nil || validPC(PREPARED_CERTIFICATE)) }
  N ← { m<RoundChange, HEIGHTm , ROUNDm, PREPARED_CERTIFICATE> ∈ M : PREPARED_CERTIFICATE != nil }
  if |N| > 0 {
    return 2f+1 <= |M| <= 3f+1 &&
    ∃ m<RoundChange, *, *, PREPARED_CERTIFICATEm> ∈ N : validPC(PREPARED_CERTIFICATEm) && PCValue(PREPARED_CERTIFICATEm) = V && 
	∀ n<RoundChange, * , *, PREPARED_CERTIFICATEn> ∈ N != m : PCRound(PREPARED_CERTIFICATEm) >= PCRound(PREPARED_CERTIFICATEn)
  }
  return 2f+1 <= |M| <= 3f+1 &&
}
```

#### quorumRound

Asserts that at least 2f+1 round change messages share the same round and
returns that round.
```
quorumRound() {
  M ← { m<RoundChange, CURRENT_HEIGHT, ROUNDm, *>, n<RoundChange, CURRENT_HEIGHT, ROUNDn, *> : ROUNDm = ROUNDn } &&
  |M| >= 2f+1 &&
  ∃ R : ∀ m<*, *, ROUNDm, *> ∈ M : ROUND = ROUNDm
  return R
}
```

#### f1Round

Asserts that there are at least f+1 round change messages and returns the
lowest round from the top f+1 rounds.
```
f1Round() {
  // This is saying that for any ROUNDm there cannot be >= f+1 elements set with a
  // larger ROUNDm, since if there were that would mean that ROUNDm is not in the top
  // f+1 rounds. 
  M ← { m<RoundChange, CURRENT_HEIGHT, ROUNDm, *> : |{ n<RoundChange, CURRENT_HEIGHT, ROUNDn, *> : ROUNDm < ROUNDn }| < f+1 } &&
  |M| >= f+1 &&
  ∃ R : ∀ m<*, *, ROUNDm, *> ∈ M : R <= ROUNDm
  return R
}
```

#### onRoundChangeTimeout
As long as the round and height have not changed since it was scheduled
onRoundChangeTimeout sets the desired round to be one greater than the Current
round, sets the current state to be WaitingForNewRound and broadcasts a round
change message.

Note: This function is referred to in the code as
`handleTimeoutAndMoveToNextRound`, which is misleading because it does not move
to the next round, it only updates the desired round and sends a round change
message. Hence why it has been renamed here to avoid confusion.

```
onRoundChangeTimeout(HEIGHT, ROUND) {
  if H = CURRENT_HEIGHT && ROUND = CURRENT_ROUND {
    DESIRED_ROUND ← CURRENT_ROUND+1
    CURRENT_STATE ← WaitingForNewRound
    schedule onRoundChangeTimeout(CURRENT_HEIGHT, DESIRED_ROUND) after roundChangeTimeout(DESIRED_ROUND)
    bc<RoundChange, CURRENT_HEIGHT, DESIRED_ROUND, CURRENT_PREPARED_CERTIFICATE>
  }
}
```


## Appendix 1: Notation
Elements of sets are represented with lower case letters (e.g. `m ∈ M`).
Because all sets are messages we use `m` to represent an element and if we
need to denote 2 messages from the same set we use `n` to denote the second
message. 

Composite objects are defined by a set of comma separated variables enclosed in
angled brackets.\
E.g. `<A, B, C>`

If we need to refer to the composite object we prefix the angled brackets with
an identifier that is a lower case letter.\
E.g. `m<A, B, C>`

An identifier can also be provided in order to distinguish variables belonging
to a composite object from other similarly named variables in the same scope.
In this case the variables are annotated with the identifier by appending the
identifier to the variable name.\
E.g. `m<Am, Bm, Cm>`

If all instances of a composite object element share the same value for one
of its variables then that value can be used in the definition. E.g. `<A, B, CURRENT_HEIGHT>`
represents a composite element with variables `A` `B` and current height.

If a composite object element has variables for which the value is not
important then `*` is used in the place of that variable.

### Numbers of participants
`3f+1 - the total number of participants`\
`2f+1 - a quorum of participants`
`f - the number of failed or malicious parcicipants that the system can tolerate`

### Participant states
`AcceptRequest`\
`Preprepared`\
`Prepared`\
`Committed`\
`WaitingForNewRound`

### Message composite object structures
`<Preprepare, HEIGHT, ROUND, VALUE, ROUND_CHANGE_CERTIFICATE>`\
`<Prepare, HEIGHT, ROUND, VALUE>`\
`<Commit, HEIGHT, ROUND, VALUE>`\
`<RoundChange, HEIGHT, ROUND, PREPARED_CERTIFICATE>`\
`<PreparedCertificate, M, VALUE>`

### Event composite object structures
Events are a means for the application to communicate with the consensus
instance, they are never sent across the network.

`<RequestEvent, HEIGHT, VALUE> - request event, provides the value for the proposer to propose`\
`<FinalCommittedEvent> - final comitted event, sent by the application to initiate a new consensus instance`

### Pseudocode notation
```
Function definitions are represented as follows where the name of the function is foo, its
parameters are X and Y, functions can optionally return a value.

foo(X, Y) {
  ...  
  return X
}
```
```
Conditional statements are represented as follows where statements inside only
one set of the curly braces will be executed if the corresponding condition C
evaluates to true or in the case of the final else those statements will be
executed if none of the previous conditions evaluated to true. 

if C {
  ...  
} else if C {
  ...  
} else {
  ...  
}
```
```
upon: UponCondition - Pseudocode directly following upon statements is executed
when the associated UponCondition evaluates to true

Upon statements are triggered upon receipt of messages or events when the
associated upon condition evaluates to true.
Upon conditions are structured thus:

<composite object, set of objects or eventto match against> <additional qualifications>

E.G:
// 2f+1 commit messages for the current round and height with a non nil value.
M ← <Commit, CURRENT_HEIGHT, CURRENT_ROUND, VALUE> && |M| = 2f+1 && VALUE != nil
```
```
schedule <function call> after <duration> - This notation schedules the given
function call to occur after the given duration.

```

### Math notation
`← - assignment`\
`= - is equal`\
`!= - is not equal`\
`&& - logical and`\
`|| - logical or`\
`{X, Y} - the set containing X and Y`\
`|M| - the cardinality of M`\
`m ∈ M - m is an element of the set M`\
`{ m : C(m) } - set builder notiation, the set of messages m such that they satisfy condition C`\
`∃ m : C(m) - there exists m that satisfies condition C`\
`∀ m : C(m) - all m satisfy condition C`

### Math Notation examples
```
// There exists a commit message m in M such that m's height (HEIGHTm) is
// less than m's round (ROUNDm) and m's value is not important.
∃ m<Commit, HEIGHT, ROUND, *> ∈ M : HEIGHTm < ROUNDm

// The cardinality of prepare messages in M with height and round equal
// to CURRENT_HEIGHT and value equal to VALUE is greater than 1 and less than 10.
1 < |{ m<Prepare, CURRENT_HEIGHT, ROUNDm, VALUEm> ∈ M : ROUNDm = CURRENT_HEIGHT && VALUEm = V }| < 10
```

## Strange things

This is from check message but it doesn't check the message
it just compares current round to desired

consensus/istanbul/core/backlog.go:64
  if c.current.Round().Cmp(c.current.DesiredRound()) > 0 {

## Thoughts

The future preprepare timer seems unnecessary, shouldn't the future preprepare
message simply be handled when moving to the future sequence? Actually I think
the future prepare timer is there to ensure that the network doesn't race ahead
of the proposed block times, but it would be better if nodes simply waited some
amount of time from the last block rather than making a calculation based on
the time value set by the proposer.

It seems the resetResendRoundChangeTimer functions are there to ensure round
changes get resent, but I don't think we need to represent that here because we
assumed reliable broadcast.

When a round change timeout occurs it starts the timer for the next round
change, which will result in another round change message being broadcast for
the newer round, so why do we need the resendRoundChangeMessage functionality.