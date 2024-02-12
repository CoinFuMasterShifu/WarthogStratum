# Pool Mining in WARTHOG
We cover the following:

- Stratum Protocol for Warthog
- Pool Dev Guide
- Miner Dev Guide
- Integration List
<hr/>

# Stratum Protocol for Warthog
Messages are newline (`"\n"`) terminated json strings sent over plain TCP with optional TLS encryption. URI scheme is `stratum+tcp` for plain TCP and `stratum+ssl` or `stratum+tls` if TLS encryption is used (for some reason "`ssl`" is more prevalent than "`tls`" as URI scheme despite in fact TLS is actually used). 


## Establishing a stratum connection
After establishing a TCP connection (and optionally TLS) the first message is sent from the Client and must be of type `mining.subscribe`:
### Mining.subscribe
#### Request from client:
```
{
  "id": 1,
  "method": "mining.subscribe",
  "params": [
    "MinerName/1.0.0",           # user agent / version
    "<suggested session id>"     # optional suggested session Id
  ]
}

```
#### Response from pool:
```
{
  "id": 1,                 # same as "id" in the request
  "result": [
    [
      [
        "mining.notify",
        "extra1hex"        # session ID, often just equal to the extra nonce 1
      ]
    ],
    "extra1hex",           # extranonce 1 in hex
    2                      # size of extranonce2
  ],
  "error": null
}
```

### Mining authorize
The next message is again send from the client and must have `"method"`  field equal to `"mining.authorize"`:
#### Request from client:
```
{
  "id": 2, 
  "method": "mining.authorize",
  "params": [
    "address.worker",  # user name, convention is to specify address and worker id
    "password"         # password
  ]
}
```

#### Response from pool:
```
{
  "id": 2,        # same "id" as in the request
  "result": true,
  "error": null
}
```

Now the connection is established. The pool n
## Events pushed from pool server to clients
There are two kind of events that sent from the pool: `"mining.notify"` and `"mining.set_difficulty"`.

### Mining.notify
```
{
  "id": null,
  "method": "mining.notify",
  "params": [
    "jobId",         # jobId has to be sent back on submission
    "prevHash",      # hex encoded previous hash in block header
    "merklePrefix",  # hex encoded prefix of 
    "version",       # hex encoded block version
    "nbits",         # hex encoded nbits, difficulty target
    "ntime",         # hex encoded ntime, timestamp in block header
    false            # clean? If yes, discard old jobs.
  ]
}
```
#### Notable differences from Bitcoin's Stratum protocol:
This method has only 7 parameters instead of 9 parameters in Bitcoin's stratum protocol. Bitcoin's stratum protocol's third (coinbase 1) and fourth (coinbase 2) parameters are not necessary in Warthog. Furthermore instead of Bitcoin's *Merkle branches* parameter we have a Merkle prefix parameter. Merkle root is computed as follows:

#### Merkle root computation
Merkle root is `sha256(merklePrefix + extranonce1 + extranonce2)`. Miners can choose extranonce2 arbitrarily (with length specified by pool) to generate different Merkle roots.

#### Header structure
The parameters sent in a `"mining.notify"` event and a 4-byte `nonce` selected by the miner can be used to form a header to be hashed by the miner as follows:

Bytes | Meaning
------|--------
1-32  | prevHash
33-36 | nbits
37-68 | Merkle root (controlled by miner, see above)
69-72 | version
73-76 | ntime
77-80 | nonce (chosen by miner)

### Mining.set_difficulty

```
{
  "id": null,
  "method": "mining.set_difficulty",
  "params": [
    100000     # difficulty
  ]
}
```

#### Notable differences from Bitcoin's Stratum protocol
In contrast to Bitcoin's stratum protocol the target is just the inverse of the difficulty. In Bitcoin there is an additional factor of 2^32 involved for historical reasons. Since Warthog was written from scratch, it not carry this historical burden.
This means the miner must meet the target `1/difficulty` to mine a share. 

## Events pushed from miner to pool
### Mining.submit
When the miner has found nonce and extranonce2 such that the block is 
```
{
  "id": 4,
  "method": "mining.submit",
  "params": [
    "jobId",       # jobId from mining.notify
    "0000",        # extranonce2 hex
    "a5b378fe",    # time hex (in shifupool equal to ntime from mining.notify)
    "a28a04a2"     # nonce hex
  ]
}
```

Pool will reply with
```
{
  "id": 4,        # same "id" as in the request
  "result": true,
  "error": null
}
```
or report error.
## Error reporting
Pool response must specify request id in the `"id"` field. Errors are reported by setting `"result"` to `null` and specifying error details in an array of size 3 (code, message, additional info) in the `"error"` field.

```
{
  "id": 3,
  "result": null,
  "error": [
    1,              # error code
    "error msg",    # error message
    null            # additional error information
  ]
}
```


01/21/2024 6:07 PM
Every coin has its own stratum specification and I changed just very little. Basically we do not have coinbases but mining.notify looks like this:

```
{
  "id": null,
  "method": "mining.notify",
  "params": [
    "jobId",         # jobId has to be sent back on submission
    "prevHash",      # previous hash in block header
    "merklePrefix",  # prefix of 
    2,               # version
    "nbits",         # nbits, difficulty target
    "ntime",         # ntime, timestamp in block header
    false            # clean? If yes, discard old jobs.
  ]
}
```
# Janushash

Warthog Network uses a [Proof of Balanced Work](https://github.com/CoinFuMasterShifu/ProofOfBalancedWork/blob/main/PoBW.pdf) mining algorithm. This is a completely new concept in the crypto space. It is possible to combine two or more proof of work algorithms multiplicatively. We have chosen two: Sha256t and Verushash v2.1. The specific mining algorithm that is used by Warthog is [Janushash](https://github.com/CoinFuMasterShifu/Janushash). For a block to be valid the following two conditions must be met:

1. Sha256t(header) > c for some hard-coded constant c.
2. Verushash(header)*Sha256t(header)^0.7 < t for the target t.

Hashes are interpreted as fixed-point numbers between 0 and 1. Therefore exponentiation to power 0.7 and multiplication is defined. There is some hard math behind these concepts, for example:
- Does classical retargeting work here? -> Yes.
- How to mine this thing efficiently? -> Compute Verushash (slower hashrate) only on args with smallest Sha256t > c.
- How to compute mining efficiency in terms of Sha256t and Verushash hashrate? -> [See here](https://github.com/CoinFuMasterShifu/Janushash#janusscore---a-formula-to-determine-mining-efficiency)

 The essential insight in combining hashes multiplicatively is that there is no bottleneck as in combining them via chaining different hash functions. Different devices can work in parallel to compute different hash functions on the same input. 

### Implementation:
The function to check for valid Proof of Balanced Work can be implemented as follows:
```c++
bool valid_pow(Header h, Target t)
{
    auto sha256tFloat { CustomFloat(hashSHA256(hashSHA256(hashSHA256(h)))) };
    auto verusFloat { CustomFloat(verus_hash(h)) };
    constexpr auto c = CustomFloat(-7, 2748779069); // c = 0.005
    if (sha256tFloat < c) {
        return false
    }
    constexpr auto factor { CustomFloat(0, 3006477107) }; // = 0.7 
    auto hashProduct { verusFloat * pow(sha256tFloat, factor) };
    return hashProduct < t;
}
```

I have written the `CustomFloat` class [here](https://github.com/CoinFuMasterShifu/CustomFloat). It is a portable floating point representation with math functions (`log2`, `pow2`, `pow`). It supports:
- Conversion of a hash into `CustomFloat` number representation.
- Multiplication
- Raising a number to some exponent

I needed to write CustomFloat because I do not like to use C math library functions for consensus-critical parts of a cryptocurrency. With CustomFloat everyone will get the exact same result guaranteed. I followed https://github.com/nadavrot/fast_log to implement log2 and pow2 functions (I had to adapt the constants to compute log2 instead of natural log). 

# Pool Dev Guide

## Share validation

I have written a C++ backend which support computes the hash product

- Verushash(header)*Sha256t(header)^0.7

This is a number between 0 and 1. If the first condition 

1. Sha256t(header) > c for some hard-coded constant c.

is not met it returns 1. This is a convenient way for pools to check valid shares: Instead of using the block real target, the pool would set an easier difficulty in Stratum jobs, which corresponds to a larger target $t^*$ than the block target $t$. A share is valid if the hash product returned by the backend is smaller than $t^*$. If the returned value is 1 as is done by the backend if the first condition above is not met, the share is automatically invalid because $t^*$ will be between 0 and 1.



[The pool backend](https://github.com/warthog-network/pool_backend) has [two endpoints](https://github.com/warthog-network/pool_backend/blob/master/backend.cpp#L63-L85):

1. `/score/:headerhex`: It computes the score above (hash product). For performance the score is not embedded into json encoded but directly returned. On error an empty string is returned. `headerhex` is the hex encoded header (80 bytes = 160 bytes hex encoded).
2. `/target_to_double/:targethex`: It converts the block target from byte representation to double. This is important to check if the score actually solves a block.


## Block structure
Each block consists of a 10 extranonce bytes followed by 3 sections:

#### 1. New  address section (`2+n1*20` bytes)
This section starts by encoding the number `n1` of new addresses registered in this block, then `n1` elements of `20 ` bytes each follow.

#### 2. Reward section (`16` bytes)
This section encodes the mining reward (miner + amount).

#### 3. Transfer section (`4+n3*99` bytes)
It starts by encoding the number `n3` of transfers in a 4 byte network byte order integer. Then follow `n3` entries of 99 bytes each, one for each transfer. If `n3` is 0 then the transfer section is omitted completely, in this case it is 0 bytes long. Miner devs can detect this by comparing the current cursor with the end of the block byte vector when parsing the sections of a block.


## Merkle Root Computation
Merkle tree has the `n1 + 1 + n3` elements of the three sections as its leaves: `n1` 20 byte elements, one 16 byte element and `n3` 99 byte elements. The Merkle tree uses `sha256` to combine two elements into one in each hierarchy level.

Merkle root is computed this way:

```c++
Hash BodyView::merkleRoot() const
{
    std::vector<Hash> hashes(nAddresses + nRewards + nTransfers);

    // hash addresses
    size_t idx = 0;
    for (size_t i = 0; i < nAddresses; ++i)
        hashes[idx++] = hashSHA256(s.data() + offsetAddresses + i * AddressSize, AddressSize);

    // hash payouts
    for (size_t i = 0; i < nRewards; ++i)
        hashes[idx++] = hashSHA256(data() + offsetRewards + i * RewardSize, RewardSize);
    // hash payments
    for (size_t i = 0; i < nTransfers; ++i)
        hashes[idx++] = hashSHA256(data() + offsetTransfers + i * TransferSize, TransferSize);
    std::vector<Hash> tmp, *from, *to;
    from = &hashes;
    to = &tmp;

    do {
        to->resize((from->size() + 1) / 2);
        size_t j = 0;
        for (size_t i = 0; i < (from->size() + 1) / 2; ++i) {
            HasherSHA256 hasher {};
            hasher.write((*from)[j].data(), 32);
            if (j + 1 < from->size()) {
                hasher.write((*from)[j + 1].data(), 32);
            }

            if (to->size() == 1)
                hasher.write(data(), 10);
            (*to)[i] = std::move(hasher);
            j += 2;
        }
        std::swap(from, to);
    } while (from->size() > 1);
    return from->front();
}
```

In the last step we append the 10 byte extranonce space from the beginning of the block to the hashed data for the final hash. The data we append these 10 `extranonce` to we call `merklePrefix`. The Merkle root is therefore `sha256(merklePrefix + extranonceBytes)`. The `merklePrefix` is transmitted in the Stratum protocol. It is either 32 bytes or 64 bytes long. Only if you have 1 leaf in the Merkle tree, the Merkle prefix will be 32 bytes, otherwise the Merkle prefix will be the two children of the root concatenated.

## Scanning the Chain
Refer to [Node API](https://github.com/warthog-network/Warthog/blob/master/doc/API.md). For example request block data at some height via `/chain/block/:height`, check balance via `/account/:address/balance`, check history via `/account/:address/history/:beforeTxIndex`.
## Pool Payout
We have sample code in [NodeJS](https://github.com/warthog-network/Warthog/blob/master/doc/integration_nodejs.md), [Python](https://github.com/warthog-network/Warthog/blob/master/doc/integration_python.md) and [Elixir](https://github.com/warthog-network/Warthog/blob/master/doc/integration_elixir.md) demonstrating how to craft, sign and submit a transaction to [node API](https://github.com/warthog-network/Warthog/blob/master/doc/API.md).

# Miner Dev Guide

Optimally mining janushash exhausts both, a system's CPU and GPU to their limits. GPU is more efficient at Sha256t computations while CPU is more efficient at Verushash v2.1. Since Sha256t hashrate will usually be larger than Verushash v2.1 hashrate by orders of magnitude, it is important to decide optimally on which headers we evaluate the Verushash v2.1 hash function.

Since Sha256t and Verushash v2.1 are different proper hash functions the result of one does not correlate with the result of the other (mathematically we can model the two hash functions' outcomes as *independent*). Therefore to minimize the janushash value  (which needs to be below the target to mine a block)

`verushash(header)*sha256t(header)^0.7`

it is best to evaluate `verushash` on the headers with smallest `sha256t` values that are still larger than the constant c to match the first mining condition above:

1. Sha256t(header) > c for some hard-coded constant c.

In particular out of the many GPU computed sha256t hashes we need to select the smallest that are greater than c. We select just as many as the CPU can handle. 

This gives us the following approach:

1. Mine sha256t of headers on GPU
2. Send those headers with sha256t > c and smaller than c + hr_CPU/hr_GPU into a queue that is processed by CPU to evaluate verushash on them.
3. Compute Verushash v2.1 on these headers
4. Evaluate janushash number representation `verushash(header)*sha256t(header)^0.7` and check if it is smaller than the target `t`.

Note:
* Above in 2. we use the quotient of CPU and GPU hashrates on verushash, sha256t respectively. The idea behind the band [`c`, `c+hr_CPU/hr_GPU`] is that on average, the number of sha256t hashes that fall into this band will be at rate `hr_CPU`, so this strategy will produce header candidates in the CPU queue at exactly the rate the CPU can handle.
* To evaluate janushash number representation in 4. above, we should copy not only the headers but also the first 4 bytes of the evaluated sha256t hash from GPU device to host memory. Otherwise we would need to evaluate `sha256t(header)` again.


# Integration list
## Pools
* [acc-pool](https://warthog.acc-pool.pw/)

## Miners
* [Janusminer](https://github.com/CoinFuMasterShifu/janusminer) (open source)
