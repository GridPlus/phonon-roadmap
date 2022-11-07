# Phonon - Asset Validation

Asset validation is the process of which a phonon client checks that a phonon truthfully represents a real digital asset on an external system outside the Phonon protocol. This process is necessary because users may set a descriptor with no accurate representation of a genuine digital asset. Though the Phonon protocol itself guarantees that private keys are transfering securely, it cannot guarantee the meaning of those private keys. Hence, asset validation establishes a link between the Phonon and the digital asset it represents.

The alpha testnet implementation assumes that the counterparty trusts the sending client and does not validate that the sent phonons are real represented assets. For beta implementation, this describes how asset validation allows Phonon clients to transfer without assuming they trust the counterparty.

Asset validation takes action during a phonon transfer, though, after a transaction proposal, before sending an approval transaction to the receiving client. You can read more about the [Phonon transfer proposal]("https://google.com").

## Interface

Generically, to validate a crypto asset, a function must be implemented that takes in a phonon, which should include its descriptor and returns a report on whether or not it represents a genuine asset or if they were any errors during this validation process.


For each crypto asset transfer in the system, an implementation of a validator must support each specific crypto asset. Typically, a validator will be selected using the currency type tag in the Phonon descriptor as a key. Clients may configure several validators for each specific asset or enable other schemes for plugging in asset validation implementations. A discussion of these security ramifications is below [Security Model].


Typical steps of an asset validation would be the following:

1. Recieve transaction proposal.
2. Parse the Phonon descriptor and public key from the transaction details.
3. Select the validator by checking what currency type the Phonon descriptor contains.
4. Check for validitiy of asset corresponding to phonon on external system.
5. If the asset is valid, return a success with no error, or if the asset is invalid, show results.
6. Return results to transaction approval step.

Below is pseudocode based on Go for what an asset validation implementation should look like.

```go
type ValidationResult struct {
	p *model.Phonon
	err error
}
definition AssetValidator {
    function Validate(desc *[]model.Phonon) returns []ValidationResult
}
```

The validate function takes an argument (Phonon), which must include the public key and all other required metadata necessary to find the asset on its system of origin. The response should return an error if one found, otherwise, there should be no error.

## Validating

To validate an asset, an implementation must check that the descriptor set on this phonon public key actually corresponds to a digital asset on an external system and that all the data represented in the descriptor matches the record on the external system.

For a typical digital asset, such as ETH or BTC, this involves checking that there is a corresponding address with a value equal to or greater than the denomination listed in the descriptor. For more complex assets such as NFTs or ERC-721, additional metadata, such as a contract address, may have to be included in the descriptor and used in the validation check.

What constitutes a valid asset is dependent on the particular asset, and is defined by the implementation of the asset validator.

## Security Model

Asset validation provides essential security guarantees within the phonon protocol, establishing a trustworthy link between a phonon and the digital asset it controls and represents.

The phonon secure applet can guarantee secure custody and exchange of private keys without double spending or key duplication. However, it cannot guarantee that a private key represents any actual asset. Therefore, asset validation works for hand and hand with the secure applet to handle a secure exchange between an actual on-chain assets

## ETH Validation Example

1. Get chain ID from the phonon.
2. Check the RPC endpoint that corresponds with the phonon chain ID.
3. Derive the wallet address from the public key and check that the address exists and that the on chain balance matches or exceeds the denomination listed in the phonon descriptor.
4. Lookup transaction hash on check if it matches the address on hand and the value.
