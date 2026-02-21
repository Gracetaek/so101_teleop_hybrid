# Patch Notes: Network Jitter Mitigation

During Phase 2 testing we observed that the standard LeRobot teleop loop
could abort unexpectedly with the error:

ConnectionError: Failed to sync read 'Present_Position' on ids=[1, 2, 3, 4, 5, 6] after 1 tries. [TxRxResult] There is no status packet!


This happens when the serial bus fails to return a status packet within
the very short timeout built into LeRobot’s _sync_read call. When the
follower is connected via a Wi‑Fi bridge (as in Phase 2), occasional
latency spikes or TCP stalls are unavoidable. A single missed packet
should not abort the entire session.

## Changes applied

To improve resilience, we modified lerobot/src/lerobot/motors/motors_bus.py:

**Increase retry count**: Both sync_read and _sync_read now take
num_retry: int = 5 as their default. This means up to six attempts
(initial + five retries) will be made to read a value before raising
an exception.

**Add backoff between retries**: We insert a small time.sleep(0.01)
delay after a failed txRxPacket() to allow transient network
jitter to settle before trying again.

**Update docstrings and remove duplicate setup calls**: The
documentation within the functions now reflects the new default and we
removed a redundant call to _setup_sync_reader().

**Guard against comm is None**: In the rare case that
txRxPacket() returns no result, we raise a ConnectionError with a
meaningful message.

Here’s the essential portion of the updated _sync_read:

def _sync_read(..., num_retry: int = 5, ...):
    self._setup_sync_reader(motor_ids, addr, length)
    comm = None
    for n_try in range(1 + num_retry):
        comm = self.sync_reader.txRxPacket()
        if self._is_comm_success(comm):
            break
        logger.debug(
            f"Failed to sync read @{addr=}" ...
        )
        if n_try < num_retry:
            import time
            time.sleep(0.01)
    ...

## Recommended run flags

In conjunction with the code changes, we found that lowering the teleop
frame rate reduces bus pressure and network load. Use:

lerobot-teleoperate --fps 10 ...


This sets the control loop to 10 Hz instead of the default 60 Hz. Lower
rates further (e.g. 5 Hz) if you continue to see occasional aborts.

## Outcome

After applying these modifications and running with --fps 10, teleop
across the SSH tunnel and socat bridge remained stable for extended
periods (10–15 minutes in our tests) without aborts. You may adjust
num_retry and the backoff delay to suit your network conditions.
