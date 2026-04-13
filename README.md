# Bittensor Improvement Tenets (BITs)

Welcome to the Bittensor Improvement Tenets (BITs)
[repository](https://github.com/opentensor/bits). This repository serves as the central
location for submitting, discussing, and tracking proposals for changes and improvements to the
Bittensor protocol.

## What are BITs?

A Bittensor Improvement Tenet (BIT) is a technical design document providing information to
the Bittensor community or describing new features for the Bittensor protocol and surrounding
ecosystem. A BIT should provide a convincing rationale for and a concise technical
specification of any underlying features described in the BIT. BITs are intended to be the
primary mechanism for proposing new features and changes, collecting community input on an
issue, and documenting design decisions that impact the protocol.

## Types of BITs

BITs are classified into the following categories:

- **Core:** Proposals that impact the core Bittensor protocol and consensus rules.
- **Subtensor:** Proposals that relate to the Subtensor blockchain and related functionality.
- **Networking:** Proposals related to networking protocols, node interactions, or networking
  infrastructure.
- **Interface:** Proposals related to API, CLI, or user interface improvements.
- **Meta:** Proposals about processes or changes to the BITs system itself.
- **Informational:** Proposals that provide general guidelines or information to the Bittensor
  community but do not propose a new feature.

## BITs Lifecycle

BITs pass through several stages before it becomes final:

- **Draft:** The initial state of a BIT when submitted as a pull request. In this stage, the
  BIT is open for discussion and feedback.
- **Review:** The BIT has passed the initial review and is now under formal review by the BIT
  editors and the community.
- **Last Call:** The BIT is nearing finalization and has a set period for final comments and
  objections.
- **Final:** The BIT is considered complete and implemented (or ready for implementation).
- **Stagnant:** The BIT has not been updated for a significant period or lacks consensus, so it
  is no longer considered active.
- **Withdrawn:** The author of the BIT has decided to withdraw the proposal.
- **Living:** The BIT is a living document that is continually updated with new information
  (e.g., coding standards or best practices).

## How to Submit BITs

1. **Fork this repository.**
2. **Create a new file** in the `bits/` directory named `BIT-XXXX.md`, where `XXXX` is the next
   available BIT number. You can use the [BIT template](bits/BIT-0000-template.md) as a starting
   point.
3. **Fill in the tenet** with your proposal details.
4. **Submit a pull request** with your new BIT. The title should be "BIT-XXXX: [Title]" where
   `XXXX` is your BIT number.
5. **Engage in the discussion**: Address any feedback from the community and BIT editors during
   the review process.

## BITs Numbering

BITs are assigned numbers in the order they are proposed. The number `0000` is reserved for the
BIT tenet. Once a BIT is accepted and merged, its number is locked and cannot be changed.

## Roles and Responsibilities

- **Author:** The person who wrote and is responsible for the BIT.
- **Editor:** A member of the community who is responsible for ensuring that BITs are clear,
  concise, and meet the repository's standards. Editors do not make decisions about BIT
  approval but facilitate the process.
- **Reviewer:** Any community member who reviews and provides feedback on BITs during the Draft
  and Review stages.

## Discussions

A discussion forum is provided at https://github.com/opentensor/bits/discussions for discussing
preliminary ideas and collecting informal feedback before submitting a BIT.

## FAQs

### What is the purpose of BITs?
BITs serve as the primary mechanism for proposing new features or changes to the Bittensor
protocol, fostering open-source development, transparency, and structured decision-making.

### How are BITs approved?
BITs are approved through community consensus during the Review and Last Call stages. Editors
facilitate the process but do not unilaterally approve BITs.

### Can I update BITs after they are finalized?
Once BITs are finalized, they are generally considered complete. However, living BITs are an
exception and can be updated regularly.

## Related Links

- [Ethereum Improvement Proposals (EIPs)](https://eips.ethereum.org/)
- [Bitcoin Improvement Proposals (BIPs)](https://github.com/bitcoin/BIPs)

## License

This repository and all BITs are licensed under [The Unlicense](LICENSE).

---

This repository is a work in progress, and we welcome your contributions and feedback to
improve the BITs process.
