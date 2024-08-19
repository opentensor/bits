# Bittensor Improvement Proposals (BIPs)

Welcome to the Bittensor Improvement Proposals (BIPs) repository. This repository serves as the
central location for submitting, discussing, and tracking proposals for changes and
improvements to the Bittensor protocol.

## What is a BIP?

A Bittensor Improvement Proposal (BIP) is a design document providing information to the
Bittensor community or describing a new feature for Bittensor, its processes, or its
environment. The BIP should provide a concise technical specification of the feature and a
rationale for the feature. BIPs are intended to be the primary mechanism for proposing new
features, collecting community input on an issue, and documenting the design decisions that
have gone into Bittensor.

## BIP Types

BIPs are classified into the following categories:

- **Core:** Proposals that impact the core Bittensor protocol and consensus rules.
- **Subtensor:** Proposals that relate to the Subtensor blockchain and related functionality.
- **Networking:** Proposals related to networking protocols, node interactions, or networking
  infrastructure.
- **Interface:** Proposals related to API, CLI, or user interface improvements.
- **Meta:** Proposals about processes or changes to the BIP system itself.
- **Informational:** Proposals that provide general guidelines or information to the Bittensor
  community but do not propose a new feature.

## BIP Lifecycle

Each BIP passes through several stages before it becomes final:

- **Draft:** The initial state of a BIP when submitted as a pull request. In this stage, the
  BIP is open for discussion and feedback.
- **Review:** The BIP has passed the initial review and is now under formal review by the BIP
  editors and the community.
- **Last Call:** The BIP is nearing finalization and has a set period for final comments and
  objections.
- **Final:** The BIP is considered complete and implemented (or ready for implementation).
- **Stagnant:** The BIP has not been updated for a significant period or lacks consensus, so it
  is no longer considered active.
- **Withdrawn:** The author of the BIP has decided to withdraw the proposal.
- **Living:** The BIP is a living document that is continually updated with new information
  (e.g., coding standards or best practices).

## How to Submit a BIP

1. **Fork this repository.**
2. **Create a new file** in the `BIPs/` directory named `BIP-XXXX.md`, where `XXXX` is the next
   available BIP number. You can use the [BIP template](BIP-0000-template.md) as a starting
   point.
3. **Fill in the template** with your proposal details.
4. **Submit a pull request** with your new BIP. The title should be "BIP-XXXX: [Title]" where
   `XXXX` is your BIP number.
5. **Engage in the discussion**: Address any feedback from the community and BIP editors during
   the review process.

## BIP Numbering

BIPs are assigned numbers in the order they are proposed. The number `0000` is reserved for the
BIP template. Once a BIP is accepted and merged, its number is locked and cannot be changed.

## Roles and Responsibilities

- **Author:** The person who wrote and is responsible for the BIP.
- **Editor:** A member of the community who is responsible for ensuring that BIPs are clear,
  concise, and meet the repository's standards. Editors do not make decisions about BIP
  approval but facilitate the process.
- **Reviewer:** Any community member who reviews and provides feedback on BIPs during the Draft
  and Review stages.

## BIP Workflow

The typical workflow for a BIP is as follows:

1. **Draft:** The BIP is submitted as a pull request and is open for discussion.
2. **Review:** After initial discussion, the BIP moves to the Review stage, where it undergoes
   more rigorous analysis.
3. **Last Call:** If the BIP passes the Review stage, it enters the Last Call phase for any
   final comments.
4. **Final:** The BIP is finalized, merged, and implemented (if applicable).

## FAQs

### What is the purpose of a BIP?
BIPs serve as the primary mechanism for proposing new features or changes to the Bittensor
protocol, fostering open-source development, transparency, and structured decision-making.

### How are BIPs approved?
BIPs are approved through community consensus during the Review and Last Call stages. Editors
facilitate the process but do not unilaterally approve BIPs.

### Can I update a BIP after it is finalized?
Once a BIP is finalized, it is generally considered complete. However, living BIPs are an
exception and can be updated regularly.

## Related Links

- [Ethereum Improvement Proposals (EIPs)](https://eips.ethereum.org/)
- [Bitcoin Improvement Proposals (BIPs)](https://github.com/bitcoin/bips)

## License

This repository and all BIPs are licensed under [The Unlicense](LICENSE).

---

This repository is a work in progress, and we welcome your contributions and feedback to
improve the BIP process.
