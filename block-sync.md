# Block Sync Protocol: Proposal

## Maintaining a list of available block sync servers

Config parameters:
- `advertise_block_period: Duration`, every `advertise_block_period` the sync server should advertise a block.

```rust
struct AdvertiseBlock {
    chain_id: ChainID,
    highest_committed_block_height: BlockHeight,
    signature: SignatureBytes, // over chain id and highest_committed_block_height
```

```rust
struct BlockSyncClientState {
    available_servers: HashMap<(VerifyingKey, BlockHeight)>,
    blacklist: HashMap<(VerifyingKey, Instant)>
}
```

Publishing `AdvertiseBlock` by a block sync server:
- publish `block_tree.highest_committed_block_height()` via `AdvertiseBlock` broadcasted to all peers every `advertise_block_period`.

Receiving `AdvertiseBlock` by a block sync client:
1. Check if the sender is a trustworthy sync peer, i.e., either belongs to committed validator set, or to a validator set proposed by any speculative block in the block tree.
2. Check if `advertise_block.highest_committed_block_height >= block_tree.highest_committed_block_height`, `advertise_block.sync_request_limit >= 1` and if the sender is not in the `blacklist`, if not then ignore this message.
3. Add to `available_servers`.
4. Iterate through `available_servers` to filter out servers that do not satisfy the condition in 1 (this is to make sure that 'outdated advertisements' are not there anymore).

Note:
- Need to be careful about replay attacks.

## Syncing with available servers

If block sync is triggered the sync client should do as follows:
1. Update the list of `available_servers` (by filtering out the blacklisted or outdated ones).
2. Sample a random sync peer from `available_sync_servers`.
3. Broadcast a `BlockSyncRequest` to the sync peer.
4. Wait for a `BlockSyncResponse` with non-empty blocks until `now () + block_sync_timeout`, if not received then exit.
5. On receiving a `BlockSyncResponse` process the response:
   - if any of the blocks is incorrect, then `blacklist` the sync server peer, update the `available_servers` by removing the blacklisted, and exit,
6. While the received blocks are not empty keep sending a sync request to the sync peer, waiting for a response until `now () + block_sync_timeout`, and processing the response as above.
7. Once sync is done, check if the sync server failed to fulfil its commitment (explained below), and if so then also `blacklist` the peer.


#### Commitment

If a sync server sends an `advertise_block` message, then with respect to the sync client (`self`) it *commits* to:

Sending at least `expected_blocks_total = advertise_block.highest_committed_block_height - self.highest_committed_block_height` blocks to the client over the course of one sync process (possibly multiple request-response rounds)


#### Rationale

This mechanism should disincentivise Byzantine behaviour:
1. Sync servers that send incorrect blocks will be blacklisted,
2. Making the sync attempt fail by sending less blocks than needed in the response is not very harmful and anyways unlikely to succeed, since it requires that:
   - the malicious sync server is the fastest to respond, and
   - (to pass the 2nd check in step 4) the block height that the sync server advertised must equal the sync client's `highest_committed_block_height`, which is difficult to predict at the point of sending `AdvertiseBlock`.

## Triggering block sync

### 1. Event-based sync trigger

#### Rationale

A replica should sync blocks in case it sees evidence that the quorum is ahead in a higher view. Clearly, the replica has not participated in the consensus decisions ahead so its blockchain is most likely outdated.

#### Implementation
Trigger sync on seeing an `AdvertiseQC{qc}` where `qc.view > cur_view` and the `qc` is cryptographically correct.
- or where the `qc` is cryptographically correct, but for an unknown block (can't check if it the qc correctly formed because if the block has not been executed yet, we don't know if it has associated validator set updates!).
- might need to check the view too, since we don't want to trigger sync for QCs for old blocks deleted via pruning.
- unknown block and `qc.view >= cur_view`?

### 2. Timeout-based sync trigger

#### Rationale
It may be the case that 
1. The quorum is broken, or 
2. The replica is behind on a validator set update and hence not able to validate future QCs and trigger sync as above. 

In such cases, the replica will at some point be unable to make progress, and if that happens for a sufficiently long time the replica should sync.

#### Implementation

Config parameters:
- `sync_trigger_timeout: Duration`: if progress is not made for more than `sync_trigger_timeout` then trigger sync.

Definition of progress:
- updating `highest_qc`.

To decouple block sync trigger from `HotStuff` and `Pacemaker`:
The `highest_qc` can be stored by the block sync client on seeing a cryptographically correct qc via `AdvertiseBlock`, and sync can be triggered when we don't update for a long time e.g.,

```rust
struct BlockSyncClientState {
    available_servers: HashMap<(VerifyingKey, BlockHeight)>,
    blacklist: HashSet<VerifyingKey>,
    progress_tracker: ProgressTracker
}
```

```rust
struct ProgressTracker {
    highest_qc: QuorumCertificate,
    last_updated: Instant, // or last_progress_time
}
```

Note that:
- `sync_trigger_timeout` must be a multiple of `advertise_block_period`,
- after unsuccesful sync the timer must be reset, to avoid sync cycles, especially at the start of the protcol execution, hence it is more appropriate to say that "progress = updating `highest_qc` or a sync attempt". However, having a smaller `sync_retry_timeout` after which sync is triggered again if last sync was unseccesful might be helpful, but adds complexity.
