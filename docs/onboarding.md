# Developer Onboarding

## Integration rules

1. All agent-to-agent traffic is brokered by the Orchestrator; agents never call each other directly.
2. Call order at checkout: `Inventory.validate_reservations`, then `Delivery.estimate`, then `Pricing.compute_cart`, then `Payment.charge`, then `Order.create_order`, then `Inventory.commit`.
3. Payment validates the amount against the signed Pricing snapshot (`price_version_id`), never against the Order agent.
4. Every write carries an `idempotency_key`. The Returns agent generates and persists the key for each refund.
5. On a Payment timeout, the Orchestrator checks `status_by_idempotency` before asking the user to retry.
6. If a reservation expires between payment and order creation, Order returns `CONFLICT` and the Orchestrator triggers an idempotent refund.

## Interface stubs

Start with the Inventory and Payment endpoints in `api/openapi.yaml`, then use `registry/langgraph_agents.json` for the remaining agents' endpoints and SLAs.

## Test targets for the reference prototype

- Reservation expiry: stock returns to ATP after the TTL and the cart item is flagged expired.
- Price stacking: the best non-stackable promotion wins and totals are deterministic.
- Refund idempotency: a replayed refund key returns the original result and never refunds twice.
