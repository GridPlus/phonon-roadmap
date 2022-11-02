# Overview 
This spec is intended to describe the complete process by which a phonon can be securely transferred from one phonon secure applet to another. 

The current phonon process is almost secure, but leaves some gaps, obvious and nonobvious, in the security model that would be possible for attackers to exploit. This spec is intended to fully describe the complete vision of secure phonon transfers, address all of the deficiencies of the current implementation, and result in a design that allows users to transfer phonons with 100% security. 

This should cover all the steps which are taken by the secure applet (phonon-card), the steps that need to be taken by the phonon core library to validate the transfer, as well as touching on the user interface that a user will interact with to accomplish these tasks. 

The specification in this doc covers several “features” which have previously been referred to as asset/transfer validation, proposals, invoicing, and transfer replayability. Each one of these has a specific purpose for inclusion in the spec, but the design of each one makes the most sense in the context of the overall transfer process design. 

# Use Cases
## Payments
A user wants to send a specified set of phonons (a phonon transfer packet) to a counterparty’s phonon card. The user selects these phonons through the user interface, inputs the recipient’s address, and the transfer is executed. Each party in the exchange relies only on their own client and card’s behavior to validate the transaction, the counterparty client is untrusted and the counterparty is only trusted to maintain the specific guarantees provided by the phonon applet. This spec covers everything that happens after “execution.” 

## Goals
A key promise of the phonon protocol is to enable the secure exchange of digital assets off chain, guaranteed by secure hardware. In order to make that a reality, the phonon transfer system must be carefully designed to be maximally secure and reliable, as well as ideally being private and performant. 

From a pure information perspective, a phonon transfer is essentially an end to end encrypted  transfer of data from one secure applet to another which is idempotent and 100% reliable. 

Below are the properties needed to best support a system intended to securely and reliably transfer digital assets.  

### Encryption
The private data within the phonon transfer packet must remain securely encrypted from applet to applet (card to card) in order to prevent a man in the middle from stealing phonon private keys to double spend assets. The public phonon data and the phonon transfer packet as a whole may be encrypted for additional redundant privacy guarantees or for convenience, but it is not strictly necessary.  

### Idempotency
Transfers must be receivable once and only once. It must be impossible for a phonon transfer packet to be received twice, by the same or different applet instances. This is essential as otherwise attacks replaying a phonon transfer could result in doublespends within the protocol. 

### Recoverability
Since a phonon typically serves as the sole owner of the private key controlling a digital asset, it can be treated as an exclusive physical bearer instrument. If it is lost, it’s gone for good, so a system for transferring them must be 100% reliable or else it will be terrible. We can’t just reverse the transaction on the backend like the credit card companies. 

Transfers need to be resendable from the sending card in the event the recipient does not receive it for some intransigent issue in the middle of a transfer of phonons, such as an electrical failure in a card reader, or a network disruption. The packet can be kept in a sendable state as long as the phonons within it remain:

1. Unavailable to the user of the phonon applet for other purposes (such as redeeming or sending)
2. Exclusive to the original recipient. The phonons in this packet, when replayed, must be possible to receive only by the original recipient. 

### Self-Sovereignty
The transfer process should minimize dependency on external systems outside of the communicating phonon clients and secure applets. Ideally, a secure transfer could be conducted solely between two phonon applets connected to an offline client. Dependency on decentralized systems is highly preferred to centralized ones. 

Additionally, every component of the system outside of the user’s personal card and client must be assumed to be potentially adversarial, with trust extended only with verification. For instance, a user’s client should not trust their counterparty’s phonon client or card without establishing a clear chain of trust through the counterparty’s secure applet CA signed certificate. 

This property makes the system more difficult to design than relying on guarantees of centralized services, but it also serves as a key differentiator of the system. If built with truly minimal dependencies, phonon will extend the decentralization guarantees of blockchain systems and enable low cost transactions with a much lower attack surface for fraud and other costly issues than current payment networks such as credit cards. 

# Non-Goals
1. More complex or conditional exchanges which would involve additional steps, such as atomic swaps or change making. The peer to peer transfer is complex enough, and most additional features can be implemented by just performing a second transfer in the other direction after pairing. Additional types of transactions could be implemented on the future on top of this fundamental transfer system. 
2. Asset Validation for specific assets. Asset validation is going to involve different details for each type of asset that needs to be validated. We should build just the interface for validation along with an example implementation for Ethereum. Validation for other assets such as BTC, ERC-20, and Native Phonons can be built later and would be good candidates for external contributions. 
3. Automated Approval System - We’ve previously discussed implementing features by which users could manually or automatically approve transactions to or from their cards. For this spec that is out of scope, it is assumed that the counterparties intend to participate in the transaction for the purposes of this specification. Automatic proposals and approvals can be added onto this system with few changes later. 

# Secure Phonon Transfer Design
Orchestrating a secure phonon transfer requires a specific set of steps to be accomplished in a specific manner, in order to properly leverage the security guarantees provided by multiple systems, such as the phonon-card secure applet, the phonon-client, the underlying phonon asset’s original blockchain, as well as the same stack on the counterparty’s side. 

As we know, a Phonon is essentially a data structure containing the requisite information needed to control a digital asset. Typically, this is a private key along with metadata describing how to locate the asset on its origin blockchain. Phonons have private data, typically the private key, which remains hidden from all, including the owner of the phonon, and public data, typically everything else such as the originating chain name, the description of the asset, and its on chain value. 

While the phonon-card applet itself guarantees the secrecy and security of the asset private keys, it is still necessary that care be taken when transferring phonons between counterparties to ensure that each party is sending and receiving the digital assets they intend. For example, a recipient of a phonon needs to check that the phonon they are receiving actually represents the digital assets it claims to. Since phonon cards inherently only possess information that their users have given them, the public metadata (aka phonon descriptor) for each phonon is entered by the user without any verification. The consequence of this is that phonon descriptor information is inherently insecure, it must be checked by the receiving counterparty in order to ensure its veracity. 

Essentially, the only security guarantee about a phonon that the secure applet can make is that it does in fact possess the private key corresponding to a certain public key, and that that private key has remained hidden inside of a phonon during its entire lifecycle and will continue to do so until it is redeemed. This is the core property that prevents doublespends, but it means that the client will have to take on the additional responsibility of validating that phonon assets intended for transfer are legitimate blockchain assets. 

## Conceptual Phonon Transfer Sequence
1. Bob commands his phonon application to send a set of phonons to Alice 
2. Bob’s client initiates a pairing process (INIT_PAIR) with Alice’s client to connect both cards. Assuming both are legitimate phonon cards, the two are paired. This pairing process cryptographically ensures that both counterparties are legitimate phonon cards and establishes the end to end encrypted channel for them to communicate. 
3. Bob’s card returns the list of phonon public keys about to be sent along with their descriptors.
4. Alice’s client uses the received list of phonons to look up the underlying assets on their base chain(s), determining if the asset’s existence and denomination are legitimate. If illegitimate, the transaction is aborted with a refusal from Alice’s client. If good, the transaction proceeds. Let’s call this onchain asset validation. If successful, the transaction proceeds, if not an error is returned and the transaction is canceled. 
5. Alice’s client informs her card that there is an expected transfer to receive, and includes the public keys which are part of the set. (SET_RECV_LIST) Alice’s card returns a transfer nonce and transfer salt to her client, for use in Bob’s send command. This is used to prevent replay attacks on the transfer.
6. Bob’s card executes a SEND_PHONONS and encrypts the necessary phonons along with the transfer nonce and transfer salt into an outgoing transfer packet, and sends them to Alice’s client. Bob’s card retains the encrypted transfer packet so it can be reused in the event of a transaction failure. 
7. Alice’s client takes the received transfer packet and executes a RECV_PHONONS. Receive phonons checks the following:
   1. The received phonons can be parsed and stored
   2. The unique transfer data is included and correct
      1. The recipient identity public key is included and matches. (ensuring this transfer is constructed exclusively for this recipient card
      2. The transfer nonce matches the next expected value (ensuring this is not a replayed packet)
   3. The received phonon private keys correspond to the expected phonon public keys from the earlier SET_RECV_LIST. This is done by deriving the public keys from the now visible private keys and ensuring they match the earlier list which was given to the card. 

If these are all successful, Alice’s card increments its transfer nonce, and indicates success to the client.
If a check does not succeed an error should be returned. A mismatch could mean malicious activity, or some kind of bug or misconfiguration, so information should be provided to the user on the nature of the issue so that proper next steps can be taken.
8. Alice’s client sends an acknowledgement message to Bob’s client. 
9. Upon receiving the acknowledgement, Bob’s card deletes the outgoing phonon transfer packet from its storage to free space, as it is no longer needed for transaction recovery. 

## High Level Transfer Sequence
1. Bob executes Send
2. Bob’s client initiates PAIR between Bob’s secure applet and Alice’s
3. Bob’s client calls GEN_PROP to generate a transfer proposal and sends it to Alice’s card via the card to card Secure Channel 
4. Alice’s client forwards the proposal to her card to run RECV_PROP which returns the transfer proposal data to Alice’s client
5. Alice’s client performs asset validation on the transfer proposal
6. Alice’s client executes APPR_PROP referencing the transfer proposal her client just validated. Her card returns the unique transfer data
7. Alice’s client returns the unique transfer data to Bob’s client
Bob’s client instructs his card to execute SEND_PHONONS using the unique transfer data sends the phonons from the pending proposal
8. Alice client instructs her card to execute RECV_PHONONS, which completes idempotency and asset validation checks set up in APPR_PROP
9. Alice’s client returns an ack of success to Bob’s client
10. Bob’s client sends Bob’s card an ACK_SEND to clean up the now completed phonon transfer packet 

## Security Model
The transfer process is designed this way in order to allow a user to validate the assets they are receiving from a counterparty even when that counterparty may be malicious and adversarial, malversarial, adverlicious, or any combination of the above scary words. BYZANTINE!

The process above step by step establishes trust in all of the secure properties of the counterparty that could go wrong. The following properties are cryptographically validated one by one.
The counterparty is conducting the transfer from a legitimate phonon card. (Pairing) 
The phonons which are proposed for transfer actually control digital assets with the value the recipient intends to receive. (asset validation)
The phonons which are ultimately received actually correspond to the validated phonons from the above proposal. 
The phonon transfer packet sent can be received once and only once by the recipient and the recipient only. (replay attack prevention/invoicing)

## Differences from Existing Implementation
The existing implementation handles Pairing and Sending phonons but does not handle asset validation, replay attack prevention, or transfer replayability. Basically, from the high level sequence given above, it handles step 1 and 2 and partially implements steps 7 and 8. The rest of the steps are basically completely unimplemented. 

Put another way, Pair along with a basic version of Send and Receive exist and are working, everything else is missing. The major feature ideas which can be broken apart to build this transfer topology are as follows: 

### Asset validation
Is clearly completely missing, as is the SET_RECV_LIST card command to close the loop with the card on asset validation. The current application just trusts that the counterparty is being honest that the phonons it sends actually correspond to real assets, and aren’t just made up keys where they’ve SET_DESCRIPTOR to say it’s 100 BTC or whatever. 

### Replay attack prevention/Unique Transfer Data
Is also unhandled. The Pairing process establishes an AES encrypted Secure Channel which provides a weak form of replay attack prevention, but it has the limitation that it only works with single APDU’s and may be circumventable by a dedicated attacker, so it needs to be replaced by a full blown transfer uniqueness and idempotency solution. 

FYI in the past we have referred to this feature “invoicing” but that seemed to cause lots of confusion and I’m sick of saying it, so I’ve generally referred to this as exclusive idempotent transfers or unique transfer data in this document instead. This name leaves me feeling somewhat unsatisfied so I’m open to catchy names for a specific financial transaction, I’m guessing there’s a finance term I can’t think of that is exactly this. 

### Replayability 
Also entirely unhandled. Interruptions in a sent phonon packet right now would cause total loss of Phonon, aka TLOP. I’ll define whatever acronyms I want, pronounceability be damned. 

An addendum to the above transfer sequence must be designed that can actually identify when a phonon has been lost in transit and replay it at the appropriate time. The semantics of the state held by both cards in the intermediate period must be defined, including what happens with the Pairing and the SET_RECV_LIST proposal data. 

# Areas for Further Spec Refinement
As mentioned above, the scenarios and recovery steps needed to make replayability functional need to be further defined. 

The security model proposed by the inclusion of unique transfer data need to be scrutinized closely to ensure they achieve the goal of exclusivity and idempotency. 

User interfaces communicating transfer progress and outcomes to the user need to be defined. 

A graphical sequence diagram of a phonon transfer should be produced to ease understanding and clarify steps. Ideally it should account for error modes as well as successful transfers

Behavior in the case of errors needs to be defined at each step in the sequence
# Open Questions
Does the transfer proposal need to include a signature attesting that the card owns these public keys, or is that validation already handled via pairing and asset validation? If the signature is necessary, does there need to be some sort of timestamp/nonce/salt to prevent replaying packets with phonons the card no longer owns? 

# Definitions
Phonon - A data structure representing a digital asset with all of the necessary information to redeem or spend it. Includes two types of data, private and public. The secure applet guarantees that any private information is never revealed as long as the data structure remains intact within the phonon protocol, while public information is freely shared and available. Phonon private data is typically a private key, while phonon public data is typically an identifier for an origin blockchain, and other metadata describing the asset. 

Phonon Private Data - Private phonon data is generated on the secure applet and can only be revealed in specific, controlled ways. It can be sent to another phonon card via a phonon transfer (SEND_PHONONS), or it can be revealed to the user via a redemption (DESTROY_PHONON.) Outside of these scenarios, generated phonon private data is never shared outside of the original card that generated, providing a key protection against double spends within the system, as phonon private keys are never known outside of phonon secure applets except through redemption, at which point the information is taken outside of the phonon protocol. 

Phonon Public Data - Public phonon data is set via a user’s SET_DESCRIPTOR command to the card. It typically consists of metadata used to locate the digital asset, and cannot be verified by the secure applet itself, since it relies on an external system such as a blockchain to serve as a source of truth. This information can be securely shared publicly without loss of security, though it may leak certain privacy details if shared too freely.  

Phonon Transfer Packet - A packet of encrypted information comprising a set of phonons intended to be sent to a counterparty phonon secure applet. Generated by the SEND_PHONONS command, this packet is encrypted using the Secure Channel established with the counterparty during Pairing. 

Pairing - The cryptographic exchange by which two phonon cards verify each other’s legitimacy and establish a channel of encrypted communication between themselves. Specifically, the fact that a pairing has been established guarantees that the counterparty has a legitimate certificate signed by a matching certificate authority. Messages sent between cards after pairing will use the encrypted secure channel established by pairing. 

Legitimacy - In the context of the Phonon protocol, a legitimate secure applet is one which has a valid signed certificate from a trusted Phonon Certificate Authority. 

Asset Validation - The process by which a phonon client validates that a phonon actually corresponds to an on chain asset. Typically, this involves deriving an address from the phonon’s public key and comparing its stated denomination with the denomination of the corresponding onchain asset. The asset validation process is completed by the recipient phonon card confirming that the phonon private keys it has received in a transfer correspond to the public keys which were validated during the transfer proposal. 

Unique Transfer Data - The data that a phonon secure applet generates to ensure that received phonon transfers are idempotent and exclusive. This means that the transfer packet can be received once and only once and only by the single secure applet for which it was intended. This data likely must include the recipient identity public key and a transfer nonce. 
