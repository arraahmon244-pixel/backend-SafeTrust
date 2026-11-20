# Agent Bookings — Issue #778 Investigation

**Status:** Investigation complete; implementation not started
**Date:** 2026-11-20
**Repository:** safetrustcr/backend-SafeTrust

## 1. Objective

Implement agent bookings with separate booking-fee and escrow-deposit payments.

- Booking fee: 0.10 USDC through x402, payable to the SafeTrust fee account.
- PR #781's hold flow currently charges a separate 0.01 USDC x402 fee. The relationship between these fees must be clarified to prevent duplicate or unintended charges.
- Deposit: server-quoted amount funded directly into Trustless Work escrow by the agent's wallet.
- The deposit must not be charged through x402.
- The reservation must only become funded after reconciliation observes the on-chain balance.

## 2. Branch and dependencies

- PR #781 (`feat/777-booking-holds`) is open and blocked.
- PR #781 supplies the booking-hold foundation.
- Current working branch: `feat/agent-bookings-on-holds`, based on `upstream/pr-781`.
- Original branch `feat/agent-bookings` remains preserved.
- PR #769 introduces reconciliation-driven lifecycle transitions.
- PR #782 changes transaction confirmation and authentication behavior.
- These PRs are not assumed to be merged or stable. Their integration must be checked before finalizing the implementation.

## 3. Verified database findings

- The schema migration `1779300000001_migrate_to_safetrust_schema` moves `public.reservations` into `safetrust.reservations`.
- PR #781 creates `safetrust.booking_holds`.
- Both tables are in the same PostgreSQL database and schema.
- The reservation table has no `guests` column in the inspected migration and Hasura metadata.
- Reservation status values include `pending`, `escrow_created`, `funded`, and `cancelled`.
- The reservation `total_amount` is a pricing snapshot; it must not automatically be treated as the escrow deposit amount.

## 4. Proposed booking transaction

The hold conversion and reservation creation should be atomic:

1. Begin a database transaction.
2. Lock the requested hold row.
3. Validate tenant, owner, active status, and expiry.
4. Validate that the hold has not already been converted.
5. Create the reservation using the hold's validated property and date details.
6. Mark the hold as converted.
7. Commit both changes together.

If either database operation fails, roll back both.

Trustless Work calls cannot participate in the PostgreSQL transaction. Escrow deployment therefore needs an idempotent, recoverable workflow for failures occurring after the database transaction commits.

The implementation should use the existing transaction utilities in `webhook/src/services/db.ts` and follow repository conventions.

## 5. Payment and escrow separation

The booking fee and deposit are different payment obligations.

- x402 validates the 0.10 USDC booking fee.
- The deposit must be quoted server-side.
- The agent's registered Stellar wallet should be used as the funding wallet; do not trust an arbitrary wallet address supplied by the client.
- The deposit funding transaction should be prepared as an unsigned XDR for the agent to sign.
- Reservation funding status must be driven by authoritative reconciliation of observed escrow state, not merely by XDR creation or submission.

## 6. Existing integration points inspected

- Booking holds: `webhook/src/repositories/hold.repository.ts` in PR #781.
- Hold route: `webhook/src/routes/v1/bookings/hold.handler.ts` in PR #781.
- Reservation creation: `webhook/src/services/reservation.service.ts`.
- Reservation persistence: `webhook/src/repositories/reservation.repository.ts`.
- Transaction utility: `webhook/src/services/db.ts`.
- x402 middleware: existing escrow initialization route and payment middleware.
- Trustless Work client: `webhook/src/clients/trustless-work.client.ts` in PR #780.
- Stellar client: `webhook/src/clients/stellar.client.ts` in PR #780.
- Reconciliation: `webhook/src/services/reconciliation.service.ts` in PR #769.
- Transaction confirmation: `webhook/src/services/confirmation.service.ts` in PR #782.

## 7. Open technical questions

These points must be resolved before implementing the corresponding integration:

1. Which Trustless Work endpoint or supported client operation deploys the booking escrow?
2. How should the unsigned deposit-funding XDR be generated?
3. Does the shared deposit quote service from issue #19 exist on another branch or is it still to be implemented?
4. Which wallet signs escrow deployment, and which wallet signs deposit funding?
5. Can the existing confirmation flow support agent-signed XDRs without bypassing reconciliation-owned funding transitions?
6. What is the required idempotency behavior when the same hold is submitted more than once?
7. How should an escrow be handled if the hold expires before the agent signs or submits the funding transaction?
8. Is PR #781's 0.01 USDC hold fee additional to, credited toward, or replaced by issue #778's 0.10 USDC booking fee?
9. How should failed or repeated Trustless Work deployment attempts be recovered without creating duplicate escrows?

Do not invent endpoint names, response fields, or signing behavior to fill these gaps.

## 8. Proposed implementation sequence

1. Confirm the dependency and branch strategy with PR #781.
2. Implement and test atomic hold conversion plus reservation creation.
3. Implement the server-side deposit quote integration.
4. Integrate escrow deployment and unsigned funding XDR generation.
5. Integrate transaction submission and confirmation.
6. Verify reconciliation controls the final funded status.
7. Add idempotency, expiry, failure-path, and testnet integration tests.
8. Document the public API in `docs/api/agents/bookings.md`.

## 9. Safety invariants

- Never charge the deposit through x402.
- Never accept an unverified client-supplied deposit amount.
- Never mark a reservation funded just because an XDR was generated or submitted.
- Never convert an expired, inactive, or another user's hold.
- Never create duplicate reservations for the same hold.
- Preserve the existing escrow lifecycle and tenant-isolation rules.
