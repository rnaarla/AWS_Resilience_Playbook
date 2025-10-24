# Chaos Simulation Example

Run automated race condition tests weekly using this model:

1. Deploy three Enactors with intentional latency skew.
2. Simulate cleanup and update operations concurrently.
3. Verify that quorum consensus prevents active DNS plan deletion.
