# Stateless Blockchain Experiment

## Disclaimer
This repository is purely experimental and is not intended for use in production. The design choices made in this project
are impractical from both a security and usability standpoint. Additionally, the following code has not been checked for
correctness, style, or efficiency.

## Testing the Project

In the root directory:

* To build the project, run `cargo build --release`
* To start the chain, run `./target/release/stateless-blockchain --dev --execution-block-construction=Native`
* If you need to reset the chain, run `./target/release/stateless-blockchain purge-chain --dev`
* If you would like to execute tests, run `cargo test -p stateless-blockchain-runtime --release`

In the accumulator-client directory (you must use nightly Rust):

* Run `wasm-pack build` to compile the crate to WASM.
* Inside "pkg", run `npm link`

In the client directory:

* Run `npm install` to install all necessary dependencies.
* Run `npm link accumulator-client` to link to the generated WASM files.
* Run `yarn start` to start a local server on localhost:8000.

## Overview and Background
This project implements a UTXO-based stateless blockchain on Substrate using an RSA accumulator. A cryptographic accumulator is
a short commitment to a set that includes small membership proofs(a Merkle Tree is a type of accumulator). However, compared
to Merkle Trees, accumulators have constant size inclusion proofs, can efficiently add and delete(dynamic), and can support
both membership and non-membership proofs(universal).

In this scheme, validating parties only need to track a single dynamic accumulator instead of storing the entire state(such
as the UTXO set in Bitcoin). Similarly, users only need to store their own UTXOs and membership proofs. However, since users
must constantly watch for updates to the accumulator in order to update their proofs, a data service provider who handles
witnesses on behalf of users is likely to be used in practice. This particular implementation includes batching and aggregation
techniques from this paper: https://eprint.iacr.org/2018/1188.pdf.

## Implementation Details

### Setup
The accumulator can be instantiated with a group of unknown order such as an RSA group. Any RSA number up to RSA-2048
is supported(see the "accumulator" crate root). Accumulators can be instantiated with no trusted setup using class groups
but are mostly a research topic at the moment.

### Mechanics
The workflow of a stateless blockchain is as follows:

1. A mechanism like Proof-of-Work or some other minting mechanism can be used to add new coins to the accumulator.
2. To spend a coin, users construct transactions that include their UTXO, the membership witness for that UTXO, and a
valid transaction output.
3. At the finalization of a block, the miner/validator aggregates all of the inclusion proofs from the spent UTXOs and
batch deletes them from the accumulator. Similarly, the miner/validator batch adds the newly created UTXOs to the accumulator.
At each step, the miner/validator outputs a proof of correctness that the deletion/addition was executed correctly.

### Structure
The base of this project is a simple Substrate runtime. However, the core accumulator logic is stored in the "accumulator"
crate and includes all of the integer specific functions, succinct proofs of exponentiation, and all of the
functions for creating or updating membership witnesses.

The front-end for this project is stored in the "client" directory and implements a simple React page based on the
Substrate Front-End template.

Lastly, "accumulator-client" is a wrapper crate for "accumulator" that uses wasm-bindgen to export several accumulator
functions to WASM. This is necessary so that the front-end can interact with the accumulator.

## Limitations

Since this is an experimental project, there exists numerous limitations.

* Instead of integrating a Proof-of-Work module, this runtime allows users to trivially mint new coins.
* The "UTXOs" that are created more closely resemble non-fungible tokens and are not explicitly value bearing(only contain
identifier and owner).
* Users can only submit one transaction per block and each transaction is limited to one input and one output.
* Instead of aggregating inclusion proofs in memory, the "blockchain" must temporarily write the details of each incoming
transaction to storage (but are erased at the end of the block). This is currently the only viable method for processing
incoming extrinsics without modifying Substrate itself.
* The scheme does not verify any signatures. Signatures could be aggregated within a block using BLS signatures.

##  Miscellaneous

The primary computational bottleneck occurs when a UTXO is hashed to a prime representation. Although this implementation
uses a deterministic variant of the Miller-Rabin primality test that is "efficient", it is still a limiting factor.
Page 24 of https://eprint.iacr.org/2018/1188.pdf presents a modification to the inclusion proofs such that the verifier
only needs to perform one round of primality checking instead of rederiving the hash representation(which involves about log(lambda) rounds).
If transactions are taking too long to process, the block time can be modified by changing "MinimumPeriod" in the crate
root of the runtime.

With regard to semantics, it is important to note that this implementation is not *actually* a stateless blockchain since
the runtime still utilizes the underlying storage trie of Substrate as well as multiple SRML components. However, the
storage requirements are still fairly minimal.

## Future Work

Here is a non-comprehensive list of potential future steps.

* Implementing more complex UTXO logic.
* Integrating a Proof-of-Work module.
* Creating a UX friendly front-end.
* Creating a data service provider.
* Investigating class groups.
* Signature aggregation.
* Implementing vector commitments for an account-based model.
* Explore accumulator unions and multiset accumulators.

Note: This repository will no longer be maintained by its original owner after November 2019.