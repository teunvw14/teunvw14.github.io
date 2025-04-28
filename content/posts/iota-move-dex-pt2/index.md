+++
title = "Move on IOTA Rebased: Building a Decentralized Exchange Smart Contract. Part 2: Liquidity Providers and Fees"
date = 2025-04-25
+++

# 1. Introduction

This article is the second in a three-part series on building a decentralized exchange with Move on the IOTA Rebased network. In this series, we develop a Move smart contract for a DEX based on the **Liquidity Book** (LB) model.

In the previous article, we dove into how the LB model executes swaps between any token pair `L` and `R`. We set up functions to create LB pools and execute swaps inside them. This was an important first step, but our smart contract wasn't yet finished: Liquidity Providers, a crucial element of any DEX protocol, were still missing. That's what we will be adding in this article.  

This smart contract expansion is split into two parts. The first half discusses how to add liquidity provision (supplying and withdrawing liquidity). The second half discusses how to integrate fees into the smart contract, which earn liquidity providers money over time.

## 1.1 Article Series 

Though there are many beginner tutorials for Move, intermediate tutorials can be harder to find. Having familiarized themselves with the basics, developers often wonder: How do I actually build something with this? The scarcity of intermediate tutorials is unfortunate, because they can really help bridge the gap between basic understanding and practical application. This article series is an attempt to fill that gap. It tries to do this by walking through an example of building a relatively complex Move application: a decentralized exchange. The building of this smart contract is split up into three parts:

1. The [first part](../iota-move-dex-pt1/) walks through the first steps of writing a Move smart contract implementing the LB DEX model.

2. The second part (this one) describes how the DEX smart contract can be expanded to include fees and Liquidity Providers. 

3. The [third part](../iota-move-dex-pt3/) dives into *testing* Move smart contracts, which is crucial in gaining confidence that your Move does what it's supposed to do. 

## 1.2 Before We Go On

This article builds upon the first article in this series. Make sure to work through it (or at least read it) before you dive into this one, otherwise this article will not make much sense. 

In this article, we expand on the code from the previous article. If you want to be sure that you're starting off with the right code, you can copy it from [the pt-1 branch of the reference implementation on GitHub](https://github.com/teunvw14/move-liquidity-book/tree/pt-1).

In this article, we will make some major changes to the DEX smart contract. This will cause some of the tests that we wrote in part 1 (and any tests you may have written yourself) to fail. We will fix these issues, and also expand greatly on the suite of tests in the next article. 

## 1.3 Acknowledgements 

*Many thanks to [iotalabs](https://iotalabs.io/) for supporting this article series with a grant.*

# 2. Liquidity Providers

We will start by adding functionality for supplying and withdrawing liquidity. To simplify the process, we imagine that fees don't exist at all. We will just focus on creating functions to supply and withdraw liquidity to an LB `Pool`. 

## 2.1 Doing Some Accounting

We want to expand our smart contract so that anyone can provide liquidity in a `Pool`. Obviously, if someone deposits liquidity in a `Pool`, they will at some point want it back. In conclusion: we need to keep track of who is providing liquidity in a pool. 

For many DEXs, this is done through the use of *LP Tokens*. These LP Tokens directly represent a share of the total pool liquidity. When you have 1 LP token for a pool that has 100 LP Tokens in total, when you withdraw, you get 1/100th of the total pool funds. 

However, using LP Tokens only works when all the liquidity of a pool is mixed together, like in a constant-product liquidity pool. We are using Liquidity Book pools, where liquidity is distributed over discrete bins, which means that we will want to return liquidity to a liquidity provider only from the bins where they provided it. So, we will refine the LP Token model to keep track of provided liquidity.

## 2.2 Money Back Guarantee (As Long As You Have Your Receipt) - Defining The `LiquidityProviderReceipt` Type

To keep track of provided liquidity, we will give LP's a `LiquidityProviderReceipt`, which keeps a per-bin record of how much liquidity they provided. This will still function much like LP Tokens: When an LP wants to get back their liquidity, they can redeem this receipt for the provided liquidity plus any earned fees. The big difference with LP Tokens is that receipts only return liquidity from bins where it was provided. 

Let's go to our file `liquidity_book.move` and define the `LiquidityProviderReceipt` type:

```rust
public struct LiquidityProviderReceipt has key {
    id: UID,
    pool_id: ID, // The id that the liquidity was provided in
    deposit_time_ms: u64, // Timestamp from the moment the liquidity was provided
    liquidity: vector<BinProvidedLiquidity> // A per-bin record of how much liquidity was provided
}
```

We keep a `pool_id` in the receipt so that we can make sure that on withdrawal, the receipt was not meant for some other pool. 

The `deposit_time_ms` field will be used when distributing fees on withdrawal. We want to make sure that LP's don't earn fees from trades that happened before they provided liquidity. 

The last field `liquidity` keeps track of in which bins liquidity was provided and how much. We will define a custom type `BinProvidedLiquidity` representing the liquidity in one bin.  

We define the `BinProvidedLiquidity` above the `LiquidityProviderReceipt` type as follows:

```rust
public struct BinProvidedLiquidity has store, copy, drop {
    bin_id: u64,
    left: u64,
    right: u64
}

public struct LiquidityProviderReceipt has key {
        id: UID,
        pool_id: ID, // The id that the liquidity was provided in
        deposit_time_ms: u64, // Timestamp from the moment the liquidity was provided
        liquidity: vector<BinProvidedLiquidity> // A record of how much liquidity was provided
    }
```

You might wonder why we need the `BinProvidedLiquidity` type, given that it's basically a wrapper around a triple of `u64`s. Why can't we use `liquidity: vector<(u64, u64, u64)>`? This is because tuples aren't first-class values, as explained [here in The Move Reference](https://move-book.com/reference/primitive-types/tuples.html). This means, among other things, that tuples can't be used to instantiate generics. Note that even if using tuples was possible here, using the custom type strongly improves the readability of some of the code we will write later.

One final note on `LiquidityProviderReceipt`: We purposely don't give it the `store` ability so that receipts cannot (accidentally) be transferred, since an LP would then completely lose access to their liquidity.

## 2.3 Printing Receipts, or Providing Liquidity

Now, let's define a function to provide liquidity and return an instance of this new `LiquidityProviderReceipt` type. We will extract the code for distributing the liquidity across bins from the `new` function, and put this logic into a function `provide_liquidity_uniformly`. We will make just a few small changes. We will not go over this code in detail since it's covered in some detail in the previous article. The primary addition here is the creation of the `LiquidityProviderReceipt` and adding a `BinProvidedLiquidity` entry in the `liquidity` vector every time we add liquidity to a bin. These changes are denoted with `!NEW!` in the comments of the code to make it easy to spot them.

```rust
/// Add liquidity to the pool around the active bin with a uniform distribution
/// of the tokens amongst those bins.
entry fun provide_liquidity_uniformly<L, R>(
    self: &mut Pool<L, R>,
    bin_count: u64,
    mut coin_left: Coin<L>,
    mut coin_right: Coin<R>,
    clock: &Clock,
    ctx: &mut TxContext
) {
    // An odd number of bins is required, so that, including the active
    // bin, there is liquidity added to an equal amount of bins to the left
    // and right of the active bins
    assert!(bin_count % 2 == 1, EEvenBincount);

    // !NEW!
    // Assert some minimal amount of liquidity is added
    assert!(coin_left.value() > 0 || coin_right.value() > 0,
        ENoLiquidityProvided);

    let active_bin_id = self.get_active_bin_id();
    let bins_each_side = (bin_count - 1) / 2; // the amount of bins left and right of the active bin
    let bin_step_price_factor = ufp256::from_fraction((ONE_BPS + self.bin_step_bps) as u256, ONE_BPS as u256);

    // !NEW!
    // Create receipt that will function as proof of providing liquidity
    let mut receipt = LiquidityProviderReceipt {
        id: object::new(ctx),
        pool_id: self.id.to_inner(),
        deposit_time_ms: clock.timestamp_ms(), // timestamp with `Clock`
        liquidity: vector::empty()
    };

    // Add left bins
    let coin_left_per_bin = coin_left.value() / (bins_each_side + 1);
    let mut new_bin_price = self.get_active_price().div(bin_step_price_factor);
    1u64.range_do_eq!(bins_each_side, |n| {
        // Initialize new bin
        let new_bin_id = active_bin_id - n;
        self.add_bin(new_bin_id, new_bin_price);
        let new_bin = self.get_bin_mut(new_bin_id);

        // Add balance to new bin
        let balance_for_bin = coin_left.split(coin_left_per_bin, ctx).into_balance();
        new_bin.balance_left.join(balance_for_bin);

        // !NEW!
        // Update receipt
        receipt.liquidity.push_back(BinProvidedLiquidity{
            bin_id: new_bin_id,
            left: coin_left_per_bin,
            right: 0
        });
        new_bin_price = new_bin_price.div(bin_step_price_factor);
    });

    // Add right bins
    let coin_right_per_bin = coin_right.value() / (bins_each_side + 1);
    let mut new_bin_price = self.get_active_price().mul(bin_step_price_factor);
    1u64.range_do_eq!(bins_each_side, |n| {
        // Initialize new bin
        let new_bin_id = active_bin_id + n;
        self.add_bin(new_bin_id, new_bin_price);
        let new_bin = self.get_bin_mut(new_bin_id);

        // Add balance to new bin
        let balance_for_bin = coin_right.split(coin_right_per_bin, ctx).into_balance();
        new_bin.balance_right.join(balance_for_bin);

        // !NEW!
        // Update receipt
        receipt.liquidity.push_back(BinProvidedLiquidity{
            bin_id: new_bin_id,
            left: 0,
            right: coin_right_per_bin
        });
        new_bin_price = new_bin_price.mul(bin_step_price_factor);
    });

    // Add remaining liquidity to the active bin
    let amount_left_active_bin = coin_left.value();
    let amount_right_active_bin = coin_right.value();
    let active_bin = self.get_active_bin_mut();
    active_bin.balance_left.join(coin_left.into_balance());
    active_bin.balance_right.join(coin_right.into_balance());

    // !NEW!
    // Update receipt for liquidity provided in the pool.active_bin
    receipt.liquidity.push_back(BinProvidedLiquidity{
        bin_id: self.get_active_bin_id(),
        left: amount_left_active_bin,
        right: amount_right_active_bin
    });

    // Give receipt
    transfer::transfer(receipt, ctx.sender());
}
```

If you copy-pasted the above code into your project, you will see some errors. Let's fix these:

- `Clock` is not a recognized type. Import it at the top of your file with `use iota::clock::Clock`. (We use the `Clock` for timestamping receipts.)
- `ENoLiquidityProvided` is not recognized since it's a new error. Add it in the same way as the errors we defined previously with a relevant error message (like 'No tokens supplied.'). We throw this error to prevent creating a receipt when no liquidity is provided.
- The functions `get_bin_mut` and `add_bin` are not yet defined. Add these to your `liquidity_book` module. The definitions are shown below.

```rust
/// Private mutable accessor for pool bin with id `id`.
fun get_bin_mut<L, R>(self: &mut Pool<L, R>, id: u64): &mut PoolBin<L, R>{
    self.bins.get_mut(&id)
}

/// Add a bin to a pool at a particular price if it doesn't exist yet.
fun add_bin<L, R>(self: &mut Pool<L, R>, id: u64, price: UFP256) {
    if (!self.bins.contains(&id)) {
        self.bins.insert(id, PoolBin {
            price: price,
            balance_left: balance::zero<L>(),
            balance_right: balance::zero<R>()
        });
    };
}
```

Now that we have separate logic for adding liquidity to `Pool`s, we should probably remove it from the `new` function. By removing any references to initial liquidity, the `new` function simplifies to this:

```rust
/// Create a new Liquidity Book `Pool`
entry fun new<L, R>(
    bin_step_bps: u64,
    starting_price_mantissa: u256,
    ctx: &mut TxContext
) {
    let starting_price = ufp256::new(starting_price_mantissa);
    let starting_bin = PoolBin {
        price: starting_price,
        balance_left: balance::zero<L>(),
        balance_right: balance::zero<R>(),
    };
    // Start the first bin with ID in the middle of the u64 range, so as the
    // number of bins increase, the ID's don't over- or underflow
    let starting_bin_id = MID_U64;
    let mut bins = vec_map::empty();
    bins.insert(
        starting_bin_id,
        starting_bin
    );

    // Create and share the pool
    let pool = Pool<L, R> {
        id: object::new(ctx),
        bins,
        active_bin_id: starting_bin_id,
        bin_step_bps,
    };
    transfer::share_object(pool);
}
```

## 2.4 Getting Your Money Back

Now, let's look at how a receipt can be redeemed for liquidity. It's pretty straightforward: we loop over the `liquidity` as specified by the receipt, and collect the funds from each bin. If the bin doesn't contain the provided `L` tokens, we pay out as much `L` as we can and pay out the rest in `R` (equivalent to the remainder of `L`). And the same thing goes the other way around: If the bin doesn't contain the provided `R` tokens, we pay out as much `R` as we can and pay out the rest in `L` (equivalent to the remainder of `R`). It's a large block of code, but hopefully it's still doable to read through it (especially with the left-right symmetry). Some parts are also treated in more detail after the block of code.

```rust
/// Withdraw all provided liquidity from `pool` using a
/// `LiquidityProviderReceipt`.
entry fun withdraw_liquidity<L, R> (self: &mut Pool<L, R>, receipt: LiquidityProviderReceipt, ctx: &mut TxContext) {
    let LiquidityProviderReceipt {id: receipt_id, pool_id: receipt_pool_id, deposit_time_ms, liquidity: mut provided_liquidity} = receipt;

    // Make sure that the receipt was given for liquidity in this pool
    assert!(self.id.to_inner() == receipt_pool_id, EInvalidPoolID);

    let mut result_coin_left = coin::zero<L>(ctx);
    let mut result_coin_right = coin::zero<R>(ctx);

    while (!provided_liquidity.is_empty()) {
        let receipt_bin_liquidity = provided_liquidity.pop_back();
        let bin = self.get_bin_mut(receipt_bin_liquidity.bin_id);

        // Withdraw left liquidity
        let payout_left_amount = receipt_bin_liquidity.left + fees_earned_left;
        if (bin.balance_left.value() >= payout_left_amount) {
            result_coin_left.join(bin.balance_left.split(payout_left_amount).into_coin(ctx));
        } else {
            let remainder = payout_left_amount - bin.balance_left.value();
            result_coin_left.join(bin.balance_left.withdraw_all().into_coin(ctx));
            let mut remainder_as_r = bin.price.mul_u64(remainder);
            // Sometimes due to rounding, the bin might contain 1 RIGHT
            // 'too little', in which case `remainder_as_r - 1` is returned
            if (remainder_as_r == bin.balance_right.value() + 1) {
                remainder_as_r = remainder_as_r - 1;
            };
            result_coin_right.join(bin.balance_right.split(remainder_as_r).into_coin(ctx));
        };

        // The same withdrawal process, but for right liquidity
        let payout_right_amount = receipt_bin_liquidity.right + fees_earned_right;
        if (bin.balance_right.value() >= payout_right_amount) {
            result_coin_right.join(bin.balance_right.split(payout_right_amount).into_coin(ctx));
        } else {
            let remainder = payout_right_amount - bin.balance_right.value();
            result_coin_right.join(bin.balance_right.withdraw_all().into_coin(ctx));
            let mut remainder_as_l = bin.price.div_u64(remainder);
            // Sometimes due to rounding, the bin might contain 1 LEFT
            // 'too little', in which case `remainder_as_l - 1` is returned
            if (remainder_as_l == bin.balance_left.value() + 1) {
                remainder_as_l = remainder_as_l - 1;
            };
            result_coin_left.join(bin.balance_left.split(remainder_as_l).into_coin(ctx));
        };
    };
    provided_liquidity.destroy_empty();

    // Send the liquidity back to the liquidity provider
    let sender = ctx.sender();

    transfer::public_transfer(result_coin_left, sender);
    transfer::public_transfer(result_coin_right, sender);

    // Delete the receipt so liquidity can't be withdrawn twice
    object::delete(receipt_id);
}
```

There are a few things of note here. Let's go over them.

The first notable thing is this "one token too little". This happens sometimes when multiple multiplications and divisions are performed in sequence, causing (very small) rounding errors. This may start occurring especially when we increase the number of computations through adding fees. This is a quirk of fixed-point numbers. Under the hood, fixed-point numbers are represented by whole integers, so when you calculate a `1/3`, floor division is applied to the integers representing the `1` and `3` value, and you will get something ever so slightly smaller than `1/3`.

The next notable thing here is that, since a `LiquidityProviderReceipt` is non-drop, we have to unpack it (first line of the function) and then explicitly delete the object with `object::delete(receipt_id)`. (To be precise, unwrapping the receipt gives a `receipt_id` of type `UID` which is non-drop. Calling `object::delete` on `receipt_id` is the only way to not get an error. This enforces that the non-drop `LiquidityProviderReceipt` type can only be destroyed/deleted when done so explicitly inside our module.)

Also, at the beginning of the withdrawal function, we make sure that the `Pool` ID on the receipt and the actual `Pool` ID match up. We define a new error for this next to the other errors:

```rust
#[error]
const EInvalidPoolID: vector<u8> =
    b"Mismatched Pool ID: The Pool ID in the receipt does not match the Pool ID for withdrawal.";
```

## 2.5 Provide Away!

And that's it! Liquidity Providers can now provide liquidity to our `Pool`s. We will dive into how to confirm that this indeed works as intended using tests in the next article. For now, let's continue with another important addition to our smart contract: Fees.

# 3. It's Not Free, There's a Fee

Integrating fees seems simple at first. When a user executes a swap, just keep a percentage of  the input tokens as a fee. 

But this simplicity is deceiving: implementing fees turns out to be quite involved. The first step, charging the user extra, is really not very complicated; the difficulty of implementing fees lies in distributing them fairly to liquidity providers *after* they have been collected.

## 3.1 Fee System Properties

There are a few properties of our new fee system that most likely we will agree on are desirable:

- Fees generated should be distributed to liquidity providers in proportion to the share of the total bin liquidity.
- Fees generated need to be distributed only to liquidity providers who were providing liquidity when a trade is executed. We don't want users to be able to "hijack fees" by becoming liquidity providers *after* fees have been generated. 
- If a user swaps L for R, we would like the generated fee (paid in L) to only be paid out to R liquidity providers. This will turn out to be challenging. 

We will have to consider some things before we can decide how exactly we will implement fees.

## 3.2 Keeping Track

Let's consider some things.

### 3.2.1 Two Systems

First, let's consider how we should keep track of generated fees, and at what point in the supply-swap-withdraw pool lifecycle these fees should be distributed to LP's. Let's consider two ways.

1. Expand the `PoolBin` struct to hold a list of all the LP's and the total amount of fees they have earned. When a swap is executed, immediately calculate how much of the generated fee each LP has a right to (in proportion to the share of the liquidity each provided), and add it to their total.
2. Expand the `PoolBin` struct to hold a log of generated fees. When an LP goes to withdraw their liquidity, we loop over all generated fees and calculate what share they should get. Earned fees and provided liquidity are returned together.

The advantage of the method 1 is that you wouldn't need to keep track of individual fees, which could clog the `Pool` object when a lot of swaps are happening in it. It also has a big disadvantage, however: The swapping user pays for the computation needed to calculate how much of the paid fee goes to each LP. Additionally, this design seems less "pure". Consider that this method would require keeping information about particular LP's in the state of a pool. This is counter to the whole idea of a `LiquidityProviderReceipt`. These receipts represent an LP's ownership of a share of a pool (or pool bin), separating pool ownership out from the pool state.

### 3.2.2 Our Implementation

For the reasons described above, we will implement **method 2**. In general terms, we will implement fees like this:

- Fees are always paid by deducting part of the paid coin. The fees are kept in the balances of the bin where the fee was generated. This has the benefit of making generated fees available as trading liquidity.
- We expand the `PoolBin` struct to hold a log of all generated fees. Together with the generated fee amount, we will store the total bin liquidity, and a timestamp. The total bin liquidity helps divide generated fees fairly (in proportion to liquidity share), and the timestamp is used to prevent "fee hijacking".
- Though fees will be shared in proportion to bin share, we will not differentiate between supplying `L` or `R`. That is, a fee generated from an `L` to `R` swap is distributed to both `L` and `R` liquidity providers of that bin (instead of to just LP's that provided `R`). 

There is a good reason for this last design choice, by which we choose to abandon our third desired property. The reason is that, after all the `L` in a bin is swapped for `R`, there is no longer any way to discern `L` and `R` liquidity providers. Everyone is then an `R` liquidity provider. (Though it is technically possible to keep track of these shifts in liquidity, that would be problematic for the same reason as before: it's computationally very expensive, and the user that swaps would have to pay gas for it.) 

## 3.3 Generating Fees: Code

Now that all the design decisions are out of the way, it's time to start coding.

### 3.3.1 Structs

We start out once again by defining and modifying some structs. First and foremost: The `FeeLogEntry`:

```rust
/// Type for keeping track of generated fees inside a pool.
public struct FeeLogEntry has store, copy, drop {
    amount: u64,
    timestamp_ms: u64, // Timestamp from when the fee was generated
    total_bin_size_as_r: u64 // The amount of tokens in the bin (expressed in just one token)
}
```

As mentioned above, we want to prevent "fee hijacking". To this end we keep a timestamp on the `FeeLogEntry`. We also save the total bin size as `R` so that we can distribute the fee fairly to withdrawing LP's. The choice for `R` instead of `L` is very practical: Converting an amount of `R` to `L` requires a division, which is more likely to result in rounding errors (than the multiplication required to convert from `L` to `R`).

We will create a `FeeLogEntry` every time a fee is generated and save it in the bin where the swap happened. Let's modify the `PoolBin` struct accordingly:

```rust
public struct PoolBin<phantom L, phantom R> has store {
    price: UFP256, // The trading price inside this bin
    balance_left: Balance<L>,
    balance_right: Balance<R>,
    // Fee logs keep track of generated fees inside this bin.
    fee_log_left: vector<FeeLogEntry>,
    fee_log_right: vector<FeeLogEntry>,
    // Keeps track of the total provided tokens, without any fees:
    provided_left: u64,
    provided_right: u64,
}
```

When we create a fee log, we need to calculate the total provided liquidity in a bin (to distribute fees in proportion to bin share). Since we will be storing fees directly inside the `PoolBin` balances, we won't be able to read off the total provided liquidity from these balances. So we also keep a `provided_left` and `provided_right` variables that keep track of how much liquidity of each was provided in the bin in total.

We use a vector here to keep things simple. If you used a design like this at scale, you would be at risk of hitting the object size limit, as more and more `FeeLogEntry`s accrue inside a `Pool`. To get around this, in practice you would use a [table](https://docs.iota.org/references/framework/iota-framework/table).

The last modification to a struct is a small one: We add a `fee_bps` parameter to the `Pool`, which gives the pool fee in basis points (a basis point is 0.01%).

```rust
// liquidity_book.move 
public struct Pool<phantom L, phantom R> has key {
    id: UID,
    bins: VecMap<u64, PoolBin<L, R>>, // bins are identified with a unique id
    active_bin_id: u64, // id of the active bin
    bin_step_bps: u64, // The step/delta between bins in basis points (0.0001)
    fee_bps: u64, // The base fee for a swap
}
```

Make sure to resolve any errors that now occur for `PoolBin` and `Pool` initializations by adding the newly added fields where necessary, for example in the `new` function. Add `fee_bps` as an argument to `new` as well. The result is this:

```rust
entry fun new<L, R>(
    bin_step_bps: u64,
    starting_price_mantissa: u256,
    fee_bps: u64,
    ctx: &mut TxContext
) {
    let starting_price = ufp256::new(starting_price_mantissa);
    // Initialize PoolBin with default values
    let starting_bin = PoolBin {
        price: starting_price,
        balance_left: balance::zero<L>(),
        balance_right: balance::zero<R>(),
        fee_log_left: vector::empty(),
        fee_log_right: vector::empty(),
        provided_left: 0,
        provided_right: 0,
    };
    ... // other stuff

    // If you'd like, you can also set a maximum pool fee like this.
    // You will need to define MAX_BASE_FEE_BPS as a constant.
    let fee_bps = fee_bps.min(MAX_BASE_FEE_BPS);

    // Pool now with `fee_bps`
    let pool = Pool<L, R> {
        id: object::new(ctx),
        bins,
        active_bin_id: starting_bin_id,
        bin_step_bps,
        fee_bps,
    };
}
```

### 3.3.2 Fee Integration

Now that we have added the relevant structs, it's time to actually implement fees. 

Let's start out with the place where fees are *created*, and that is during a swap. Let's take a look at the new `swap_ltr` code. Although the function is now quite lengthy, try reading through the whole thing. All changes are once again annotated with `// !NEW!`, and have explanatory comments above them to make it easier to follow along. We will also cover some aspects of the code in more depth underneath the code block to make sure it's all clear.

```rust
// Swap `coin_left` for an equivalent amount of `R` in a `Pool`
public fun swap_ltr<L, R>(
    self: &mut Pool<L, R>, 
    mut coin_left: Coin<L>, 
    clock: &Clock, 
    ctx: &mut TxContext
): Coin<R> {
    let mut result_coin = coin::zero<R>(ctx);
    // !NEW!
    let fee_bps = self.fee_bps();

    // Keep emptying bins until `coin_left` is fully swapped
    while (coin_left.value() > 0) {
        let active_bin = self.get_active_bin_mut();

        // !NEW!
        // Calculate the absolute amount in `L` that will be paid as a fee.
        // Calculate swap output `swap_right` with the fee subtracted from
        // the input amount `swap_left`. 
        let mut fee = get_fee(coin_left.value(), fee_bps);
        let mut swap_left = coin_left.value();
        let mut swap_right = active_bin.price.mul_u64(swap_left - fee);

        // If there's not enough balance (after fees) in this bin to fulfill
        // swap, adjust swap amounts to maximum, and update fees accordingly.
        let bin_balance_right = active_bin.balance_right();
        if (swap_right > bin_balance_right) {
            swap_right = bin_balance_right;
            swap_left = active_bin.price.div_u64(bin_balance_right);
            // !NEW!
            // Fee is updated to match the new `swap_left` amount. Since the 
            // output amount of `R` is set already set, we have to apply the 
            // fee additively to the input amount, instead of using subtraction. 
            fee = get_fee_inv(swap_left, fee_bps);
            swap_left = swap_left + fee;
        };

        // Execute swap
        active_bin.balance_left.join(coin_left.split(swap_left, ctx).into_balance());
        result_coin.join(active_bin.balance_right.split(swap_right).into_coin(ctx));

        // !NEW!
        // Create instance of our new `FeeLogEntry` type and add it to the fee_log.
        active_bin.fee_log_left.push_back(
            FeeLogEntry {
                amount: fee,
                timestamp_ms: clock.timestamp_ms(),
                total_bin_size_as_r: amount_as_r(active_bin.price, active_bin.provided_left, active_bin.provided_right)
            }
        );

        // Cross over one bin right if active bin is empty after swap,
        // abort if swap is not complete and no bins are left
        if (active_bin.balance_right() == 0) {
            let bin_right_id = self.active_bin_id + 1;
            if (coin_left.value() > 0) {
                assert!(self.bins.contains(&bin_right_id), EInsufficientPoolLiquidity);
            };
            self.set_active_bin(bin_right_id);
        };
    };
    coin_left.destroy_zero();

    result_coin
}
```

Hopefully most of that makes sense. The one thing that probably warrants some deeper explanation is the application of the "inverse fee". Let's briefly investigate it. (Warning: Math ahead, feel free to skip ahead if you don't care about it). 

Let's look at it from the perspective of a user performing a swap. When you swap from `L` to `R`, and it's executed inside just one bin, the fee is incurred by paying out just a little bit less `R` than the price times your `L` amount (`fee_bps` / 100 percent less to be exact). Now, if instead you swap enough `L` to pay for all the `R` inside one bin *after fees*, then the pool should give you all the `R` from that bin. But that means the amount of `R` the pool pays out to you (from that bin) is set. Incurring the fee by paying out less `R` would leave `R` in the bin, which is not what the pool wants. So the pool has to incur the fee by *increasing* the amount of `L` paid for the set amount of `R`. The question then is: By how much should the pool increase the amount of `L` paid to give a fee equivalent to the reduction in amount of `R`?

Let `L` be the amount that a user wants to swap, and p is the price inside the trading bin. Then the amount of `R` is:

$$ R = p \cdot L \Big(1 - \frac{\texttt{fee\\_bps}}{10000}\Big) $$

Dividing both sides by the "fee factor" and the price, we get the following equivalent equation:

$$ L = \frac{R}{p\Big(1 - \frac{\texttt{fee\\_bps}}{10000}\Big)} =  \frac{R}{p}\Big(1 - \frac{\texttt{fee\\_bps}}{10000}\Big)^{-1}$$

So, the answer is that we have to increase the paid amount of `L` by a factor of:

$$ \Big(1 - \frac{\texttt{fee\\_bps}}{10000}\Big)^{-1}$$

This is exactly what is implemented in the code (`get_fee` and `get_fee_inv` are shown a bit further below).

Then finally, a small note on saving `self.fee_bps()`. We are forced to save the result of this call in a variable at the beginning of the function, instead of just calling the function when we need it. This is because `self.get_active_bin_mut` mutably borrows `self`. So the Move compiler won't allow us to immutably borrow self after that call. (Try replacing `fee_bps` with calls to `self.fee_bps()` and compiling to see the error for yourself. This behavior is inherited directly from how references work in Rust. You can read more about how the Rust references and the Rust borrow checker work [here](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)).

The code for `swap_rtl` is changed in an analogous manner, so the code is left out of this article for brevity. It should be doable to implement it yourself, but you can also just copy [the reference code on GitHub](https://github.com/teunvw14/move-liquidity-book/blob/702bf2c090ca052da1997d07cedf4ccd309e2c02/sources/liquidity_book.move#L479C1-L528C2) if you prefer.

### 3.3.3 Some More Helper Functions

There are four new helper functions that we used above. Let's quickly go over these.

We have `get_fee` and `get_fee_inv` for calculating fee amounts: 

```rust
/// Calculate a fee of `fee_bps` basis points.
public fun get_fee(amount: u64, fee_bps: u64): u64 {
    let fee_factor = ufp256::from_fraction(fee_bps as u256, ONE_BPS as u256);
    fee_factor.mul_u64(amount)
}

/// Calculate a fee of `fee_bps` basis points, but on the output of a trade: amount/(1-fee) - amount.
public fun get_fee_inv(amount: u64, fee_bps: u64): u64 {
    ufp256::from_fraction((ONE_BPS - fee_bps) as u256, ONE_BPS as u256)
    .div_u64(amount)
    - amount
}
```

We have the `fee_bps` accessor for `Pool`s:

```rust
/// Public accessor for `pool.fee_bps`.
public fun fee_bps<L, R>(self: &Pool<L, R>): u64 {
    return self.fee_bps
}
```

And lastly, we have `amount_as_r`, which transforms an amount of `L` and `R` into one equivalent amount of `R` at a given price. 

```rust
/// Calculate the value of two amounts represented as the right given price
/// `price`.
public fun amount_as_r(price: UFP256, amount_l: u64, amount_r: u64): u64 {
    price.mul_u64(amount_l) + amount_r
}
```

## 3.4 Paying Out Fees

After a providing liquidity for a while in an LB pool, an LP will eventually want to take profits, and withdraw their liquidity and earned fees. So, up next, we will update our code to pay out earned fees when an LP withdraws. We will use fee logs and `LiquidityProviderReceipt`s to make the calculations.

### 3.4.1 Updating the Withdrawal Code

Let's start out by modifying the `withdraw_liquidity` function we wrote above to pay out fees. 

Fees are paid out one bin at a time, same as the provided liquidity. We show the code for this below. To keep it simple, let's only consider the fees in the `fee_log_left` of each bin. The code for the `fee_log_right` will once again be analogous. 

```rust
entry fun withdraw_liquidity<L, R> (self: &mut Pool<L, R>, receipt: LiquidityProviderReceipt, ctx: &mut TxContext) {
    ... // setup code
    while (!provided_liquidity.is_empty()) {
        let receipt_bin_liquidity = provided_liquidity.pop_back();
        let bin = self.get_bin_mut(receipt_bin_liquidity.bin_id);

        // !NEW! 
        // Calculate liquidity deposited in this bin as `R`. Used for
        // calculating the share of fees below.
        let receipt_bin_liquidity_as_r = amount_as_r(bin.price(), receipt_bin_liquidity.left, receipt_bin_liquidity.right);

        // !NEW!
        // Calculate earned `left` fees
        let mut fees_earned_left = 0;
        let mut i = bin.fee_log_left.length();
        while (i > 0){
            let fee_log = bin.fee_log_left.borrow_mut(i-1);
            // This prevents fee hijacking. LP's should not get any fees 
            // from swaps that happened before they provided liquidity.
            if (fee_log.timestamp_ms < deposit_time_ms) {
                break
            };
            // Calculate earned fees proportional to bin share, 
            // fee_log.amount * share
            let fee = ufp256::from_fraction((fee_log.amount as u256) * (receipt_bin_liquidity_as_r as u256), fee_log.total_bin_size_as_r as u256).truncate_to_u64();
            // Add to the total
            fees_earned_left = fees_earned_left + fee;
            // Update fee log and delete it if empty
            fee_log.amount = fee_log.amount - fee;
            fee_log.total_bin_size_as_r = fee_log.total_bin_size_as_r - receipt_bin_liquidity_as_r;
            if (fee_log.amount == 0) {
                bin.fee_log_left.remove(i - 1);
            };
            // We traverse the fee log backwards to avoid deletion complexity
            i = i - 1;
        };

        ... // Analogous code for `right` fees
    };
}
```

One thing that might seem odd at first is this line:

```rust
fee_log.total_bin_size_as_r = fee_log.total_bin_size_as_r - receipt_bin_liquidity_as_r;
```

When the `total_bin_size_as_r` is first initialized, it is exactly that: the total liquidity in that bin in terms of `R`. It seems to have a function as a record, meaning that it should be immutable, and you would be forgiving for thinking that. The naming doesn't help here, but hopefully it makes more sense if you think of `total_bin_size_as_r` as "the remaining value (as `R`) from LP's that still have a claim to this fee". Then hopefully it makes sense that we have to mutate it, subtracting the `receipt_bin_liquidity_as_r` when the liquidity from that receipt is withdrawn. (If anything, this shows once again that naming things is hard.)

You may also wonder why we traverse from the end of the vector (most recent to oldest). The primary advantage of this is that, since `FeeLogEntry`s are pushed onto the vector in chronological order, we can simply stop (`break`) once we've reached a `FeeLogEntry` that is older than the `deposit_time_ms` on the receipt. (Another small advantage of this is that we don't have to deal with shifting indices. If we move forwards along a vector, and then delete an element, all elements after the deleted element will have their index decreased by one. Then we would have to leave the index unchanged instead of incrementing it by 1. Making this case distinction would make the code a little bit harder to read.)

### 3.4.2 Updating Bin State

There is one final change we need to make to `withdraw_liquidity`, which is that we need to update the `bin.provided_left` and `bin.provided_right` at the very end of the while loop. 

```rust
entry fun withdraw_liquidity<L, R> (self: &mut Pool<L, R>, receipt: LiquidityProviderReceipt, ctx: &mut TxContext) {
    ... // Setup as shown before
    while (!provided_liquidity.is_empty()) {
        ... // As shown before: payout calculations (fees + provided liquidity)

        // !NEW!
        // Update `bin.provided_left` and `bin.provided_right`
        bin.provided_left = bin.provided_left - receipt_bin_liquidity.left;
        bin.provided_right = bin.provided_right - receipt_bin_liquidity.right;
    };
}
```

And that's it! You have now added all required functionality for users to supply and withdraw liquidity, and earn fees in the process.

### 3.4.3 Full Withdrawal Code

The full code for `withdraw_liquidity`, including the fee calculations for the `fee_log_right` is shown below for reference.

```rust
entry fun withdraw_liquidity<L, R> (self: &mut Pool<L, R>, receipt: LiquidityProviderReceipt, ctx: &mut TxContext) {
    let LiquidityProviderReceipt {id: receipt_id, pool_id: receipt_pool_id, deposit_time_ms, liquidity: mut provided_liquidity} = receipt;

    // Make sure that the receipt was given for liquidity in this pool
    assert!(self.id.to_inner() == receipt_pool_id, EInvalidPoolID);

    let mut result_coin_left = coin::zero<L>(ctx);
    let mut result_coin_right = coin::zero<R>(ctx);

    while (!provided_liquidity.is_empty()) {
        let receipt_bin_liquidity = provided_liquidity.pop_back();
        let bin = self.get_bin_mut(receipt_bin_liquidity.bin_id);

        let receipt_bin_liquidity_as_r = amount_as_r(bin.price(), receipt_bin_liquidity.left, receipt_bin_liquidity.right);

        // Calculate earned `left` fees
        // Traverse `fee_log_left` backwards to avoid deletion complexity
        let mut fees_earned_left = 0;
        let mut i = bin.fee_log_left.length();
        while (i > 0){
            let fee_log = bin.fee_log_left.borrow_mut(i-1);
            if (fee_log.timestamp_ms < deposit_time_ms) {
                break
            };
            // Calculate earned fees proportional to bin share
            let fee = ufp256::from_fraction((fee_log.amount as u256) * (receipt_bin_liquidity_as_r as u256), fee_log.total_bin_size_as_r as u256).truncate_to_u64();
            fees_earned_left = fees_earned_left + fee;
            // Update fee log and delete if empty
            fee_log.amount = fee_log.amount - fee;
            fee_log.total_bin_size_as_r = fee_log.total_bin_size_as_r - receipt_bin_liquidity_as_r;
            if (fee_log.amount == 0) {
                bin.fee_log_left.remove(i - 1);
            };
            i = i - 1;
        };

        // Calculate earned `right` fees
        // Traverse `fee_log_right` backwards to avoid deletion complexity
        let mut fees_earned_right = 0;
        let mut i = bin.fee_log_right.length();
        while (i > 0){
            let fee_log = bin.fee_log_right.borrow_mut(i-1);
            if (fee_log.timestamp_ms < deposit_time_ms) {
                break
            };
            // Calculate earned fees proportional to bin share
            let fee = ufp256::from_fraction((fee_log.amount as u256) * (receipt_bin_liquidity_as_r as u256), fee_log.total_bin_size_as_r as u256).truncate_to_u64();
            fees_earned_right = fees_earned_right + fee;
            // Update fee log and delete if empty
            fee_log.amount = fee_log.amount - fee;
            fee_log.total_bin_size_as_r = fee_log.total_bin_size_as_r - receipt_bin_liquidity_as_r;
            if (fee_log.amount == 0) {
                bin.fee_log_right.remove(i - 1);
            };
            i = i - 1;
        };

        // Withdraw left liquidity
        let payout_left_amount = receipt_bin_liquidity.left + fees_earned_left;
        if (bin.balance_left.value() >= payout_left_amount) {
            result_coin_left.join(bin.balance_left.split(payout_left_amount).into_coin(ctx));
        } else {
            let remainder = payout_left_amount - bin.balance_left.value();
            result_coin_left.join(bin.balance_left.withdraw_all().into_coin(ctx));
            let mut remainder_as_r = bin.price.mul_u64(remainder);
            // Sometimes due to rounding, the bin might contain 1 RIGHT
            // 'too little', in which case `remainder_as_r - 1` is returned
            if (remainder_as_r == bin.balance_right.value() + 1) {
                remainder_as_r = remainder_as_r - 1;
            };
            result_coin_right.join(bin.balance_right.split(remainder_as_r).into_coin(ctx));
        };

        // Withdraw right liquidity
        let payout_right_amount = receipt_bin_liquidity.right + fees_earned_right;
        if (bin.balance_right.value() >= payout_right_amount) {
            result_coin_right.join(bin.balance_right.split(payout_right_amount).into_coin(ctx));
        } else {
            let remainder = payout_right_amount - bin.balance_right.value();
            result_coin_right.join(bin.balance_right.withdraw_all().into_coin(ctx));
            let mut remainder_as_l = bin.price.div_u64(remainder);
            // Sometimes due to rounding, the bin might contain 1 LEFT
            // 'too little', in which case `remainder_as_l - 1` is returned
            if (remainder_as_l == bin.balance_left.value() + 1) {
                remainder_as_l = remainder_as_l - 1;
            };
            result_coin_left.join(bin.balance_left.split(remainder_as_l).into_coin(ctx));
        };

        bin.provided_left = bin.provided_left - receipt_bin_liquidity.left;
        bin.provided_right = bin.provided_right - receipt_bin_liquidity.right;
    };
    provided_liquidity.destroy_empty();

    // Send the liquidity back to the liquidity provider
    let sender = ctx.sender();

    transfer::public_transfer(result_coin_left, sender);
    transfer::public_transfer(result_coin_right, sender);

    // Delete the receipt so liquidity can't be withdrawn twice
    object::delete(receipt_id);
}
```

# 4. Conclusion

That's all for now! If you completed all the steps from this and the previous article, congratulations, you have built your own working LB DEX smart contract! 

There was a lot of new code introduced here. If your code got a bit jumbled up, it might help to compare it with the [branch `pt-2` of the reference implementation](https://github.com/teunvw14/move-liquidity-book/blob/pt-2/sources/liquidity_book.move).

In the next article, we will write an extensive suite of tests that show that the smart contract works exactly as we intended. (Don't worry, it does. The tests have been written, just not the article.)

If you'd like to continue working on your Move skills, take a look at the challenges listed below. You can implement one of them or all of them. 

And finally, if you came all this way, thank you for reading!

# 5. Challenges

Here are some challenges to try to add to your DEX smart contract. They are listed in increasing difficulty.

- Someone might try to clog the fee log of a `Pool` by executing a large number of very small swaps. Try adding a check to the swap functions that some minimum number of tokens is traded. You can also make this a `Pool` parameter. 
- It may happen that a liquidity provider wants to add to their position. Implement a function that lets users add to a liquidity position. This function should take the same arguments as `provide_liquidity_uniformly`, and a `LiquidityProviderReceipt` in addition. It should check that the receipt is valid for the specified pool, and if so return a new receipt with the new supply totals. 
- Provided liquidity is distributed uniformly over the number of bins. Create some more functions that allow for providing liquidity with a different distribution over the bins. You can think of distributions that are maximal at the active bin, like a Gaussian distribution centered around the active bin. You might also create a function that lets the user pick how many bins they want to go left of the active bin and how many bins they want to go right of the active bin, so that liquidity providers can provide liquidity for only one of the pool coins. 
- Try introducing a variable fee that increases when a pool gets more active. Take a look at [the LFJ docs about variable fees](https://docs.lfj.gg/concepts/fees) for inspiration. You can implement the whole variable fee, or a simplified version.
