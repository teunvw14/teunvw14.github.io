+++
title = "IOTA Move: Common Object Access Control Patterns for Smart Contracts"
date = 2025-04-18
+++

When creating a smart contract with Move, a challenge that you will often have to solve, is how to restrict access to an object. This article explores some common object access control patterns. The patterns will become increasingly permissive, with each pattern allowing for more people to directly mutate the object. Let's dive right into it.

# Immutable Objects

Some things aren't meant to be changed. Think of emitted events, the metadata of a currency/Coin, protocol parameters, etc.. 

Move is designed so that objects can be mutated only in ways that are specified in the defining module. So objects are basically immutable by default. Let's take a look at an example:

```rust
module my_shared_object::my_shared_object {
    public struct MagicImmutableObject has key {
        id: UID,
        magic_constant: u64
    }

    fun init(ctx: &mut TxContext) {
        let magic_constant = 42;
        let magic_immutable_object = MagicImmutableObject {
            id: object::new(ctx),
            magic_constant
        };

        transfer::freeze_object(magic_immutable_object);
    }
}
```

By calling `freeze_object`, you prevent the object from ever being mutated. Any transactions trying to mutate `immutable_magic_object` will fail with the [InvalidObjectByMutRef error](https://github.com/iotaledger/iota/blob/522637331237ad597341df449dbd1b76cf2d1b8e/crates/iota-types/src/execution_status.rs#L265-L266).

Now the only way you can interact with the `immutable_magic_object` is by immutable reference. When you publish this package, the `MagicImmutableObject` will be created and get assigned a `UID`. You can then pass this identifier to functions that take arguments of type `&MagicImmutableObject`. Note that you still cannot use the reference to read fields from the object directly. The following code will **not** work:

```rust
module some_package::some_module {
    use my_shared_object::my_shared_object::MagicImmutableObject;

    fun some_function(magic_object: &MagicImmutableObject) {
        // This will not compile:
        let magic_constant = magic_object.magic_constant;
    }
}
```

Doing this leads to the following compilation error: "Invalid access of field `magic_constant` on the struct `my_shared_object::my_shared_object::MagicImmutableObject`. The field `magic_constant` can only be accessed within the module `my_shared_object::my_shared_object` since it defines `MagicImmutableObject`."

If you want to be able to read the magic constant, you need to write a public accessor function in `my_shared_object::my_shared_object` like this:

```rust
module my_shared_object::my_shared_object {
    ... // struct definition and init

    public fun magic_constant(m: &MagicImmutableObject): u64 {
        m.magic_constant
    }
}
```

Then you can use this function like this:

```rust
fun some_function(magic_object: &MagicImmutableObject) {
    // This now works!
    let magic_constant = magic_object.magic_constant();

    ... // do stuff with magic_constant
}
```

# Admin Pattern

Imagine you run an on-chain casino. Users can bet their IOTA tokens against the house in a variety of classic gambling games. Roulette, Blackjack, Slots, what have you. Now, unless you run an incredibly crooked casino, a gambler will inevitably get lucky, and win a bet. And so you need to pay up. In conclusion: the house needs to have IOTA tokens. 

So, your gambling smart contracts need to hold some IOTA tokens, but of course you don't want these IOTA tokens to be accessible by everyone. Only you, the owner of the casino, should be able to access the casino treasury, to both add to it and withdraw from it. This is where the **Admin Pattern** comes in. 

The **Admin Pattern** is when, on module publish, you create an `AdminCap` that you then require for any mutations of the relevant object. 

Let's take a look at some code.

```rust
module casino::house {
    use iota::iota::IOTA;
    use iota::balance::{Self, Balance};

    public struct CasinoHouse has key {
        id : UID,
        treasury: Balance<IOTA>
    }

    // This struct functions as a key to the treasury. 
    public struct AdminCap has key {
        id: UID
    }

    fun init(ctx: &mut TxContext) {
        // Create and share house
        let house = CasinoHouse {
            object::new(ctx),
            treasury: balance::zero<IOTA>()
        };
        transfer::share_object(house);

        // Create the AdminCap and send to the package publisher
        let publisher = ctx.sender();
        let admin_cap = AdminCap {
            id: object::new(ctx)
        };
        transfer::transfer(admin_cap, publisher);
    }

    // Though this function is public, by requiring the `AdminCap` as an 
    // argument, this function can only ever be successfully called by the owner
    // of the AdminCap object. 
    public fun empty_treasury(_: &AdminCap, house: &mut CasinoHouse, ctx: &mut TxContext) {
        let full_treasury_as_coin = house.treasury.withdraw_all().into_coin(ctx);
        transfer::public_transfer(full_treasury_as_coin, ctx.sender());
    }
    
    // By requiring the `AdminCap` here, you make sure that someone without an 
    // `AdminCap` doesn't accidentally deposits funds into the house treasury, 
    // where they can no longer access their funds.
    public fun deposit_treasury(_: &AdminCap, house: &mut CasinoHouse, deposit: Coin<IOTA>, ctx: &mut TxContext) {
        house.treasury.join(deposit.into_balance());
    }
}
```

Now whoever publishes this package will be the only one that can access the house treasury. But be careful: if you lose access to the address holding the `AdminCap` - you will lose access the funds within the `CasinoHouse`.

# Admin Invite Only Pattern

Say you found a few business partners to help you run your casino. For your daily operations, these business partners will also need to be able to access the casino treasury. (I hope you trust these guys!) So, you need a way to create multiple `AdminCap`s. We'll create a new `OwnerCap` capability type that lets the holder create new `AdminCap`s.

```rust
module casino::house {
    ...

    // This struct functions as a key to the treasury. 
    public struct AdminCap has key {
        id: UID
    }

    // This struct functions as authorization for creating `AdminCap` objects
    public struct OwnerCap has key {
        id: UID
    }

    // Creating new AdminCap objects requires the `OwnerCap`.
    public fun create_admin_cap(_: &OwnerCap, ctx: &mut TxContext) {
        let admin_cap = AdminCap {
            id: object::new(ctx)
        };
        transfer::transfer(admin_cap, ctx.sender());
    }

    fun init(ctx: &mut TxContext) {
        ... // Create and share CasinoHouse

        // Create OwnerCap and AdminCap and send to the package publisher
        let publisher = ctx.sender();

        let owner_cap = OwnerCap {
            id: object::new(ctx)
        };
        let admin_cap = AdminCap {
            id: object::new(ctx)
        };
        transfer::transfer(owner_cap, publisher);
        transfer::transfer(admin_cap, publisher);
    }

    // Same as before
    public fun empty_treasury(_: &AdminCap, house: &mut CasinoHouse, ctx: &mut TxContext) { ... }
    public fun deposit_treasury(_: &AdminCap, house: &mut CasinoHouse, deposit: Coin<IOTA>, ctx: &mut TxContext) { ... }
}
```

Note that one disadvantage of this is that once you've given someone an `AdminCap`, there's no way to "take it away", which might become a problem if you want to split ways with one of your business partners.

# Whitelist / Blacklist

A solution to not being able to remove access as the owner comes in a different access control method: the whitelist and the blacklist. A whitelist says who should have access. A blacklist specifies who should *not* have access. In this model, the 

```rust
module whitelist_package::whitelist_module {
    public struct AppState has key {
        id: UID,
        whitelist: vector<address>
    }

    public struct AdminCap has key {
        id: UID
    }

    fun init(ctx: &mut TxContext) {
        // Create OwnerCap and AdminCap and send to the package publisher
        let publisher = ctx.sender();
        let admin_cap = AdminCap {
            id: object::new(ctx)
        };
        transfer::transfer(admin_cap, publisher);

        // Create AppState that will hold the whitelist
        let state = AppState {
            id: object::new(ctx),
            whitelist: vector[]
        };
        
        transfer::share_object(state);
    }

    // Require AdminCap to add people to whitelist
    public fun whitelist_add(_: &AdminCap, addr: address, state: &mut AppState) {
        state.whitelist.push_back(addr);
    }

    // Require AdminCap to remove people from whitelist
    public fun whitelist_remove(_: &AdminCap, addr: address, state: &mut AppState) {
        let (address_in_whitelist, idx) = state.whitelist.index_of(&addr);
        if (address_in_whitelist) {
            state.whitelist.remove(idx);
        }
    }

    #[error]
    const EAddressNotWhitelisted: u64 = 0;

    // Require the AppState (of which only one exists) as an argument, and throw
    // an error if the calling address is not on the whitelist.
    public fun whitelist_only_func(state: &mut AppState, ctx: &mut TxContext) {
        assert!(state.whitelist.contains(&ctx.sender()), 
            EAddressNotWhitelisted);

        ... // do stuff
    }
}
```

A blacklist works almost exactly the same, except that we assert that someone is **not** on the blacklist. 
```rust
module blacklist_package::blacklist_module {
    public struct AppState has key {
        id: UID,
        blacklist: vector<address> // instead of whitelist
    }

    // All the stuff below is basically the same
    public struct AdminCap has key {...}
    fun init(ctx: &mut TxContext) {...}
    public fun blacklist_add(_: &AdminCap, addr: address, state: &mut AppState) {...}
    public fun blacklist_remove(_: &AdminCap, addr: address, state: &mut AppState) {...}

    #[error]
    const EAddressBlacklisted: u64 = 0;

    // Require the AppState (of which only one exists) as an argument, and throw
    // an error if the calling address is blacklisted.
    public fun blacklist_only_func(state: &mut AppState, ctx: &mut TxContext) {
        assert!(!state.blacklist.contains(&ctx.sender()), 
            EAddressBlacklisted);

        ... // do stuff
    }
}
```

# Invite Only Club Pattern

One other way to restrict access to certain smart contract functionality is by creating an *invite-only club*. With this access pattern, anyone who has a club membership, can "invite" new members by creating a membership card for them. Let's see this in code:

```rust
module invite_only_club::membership {
    public struct MembershipCard has key {
        id: UID
    }

    fun init(ctx: &mut TxContext) {
        let publisher = ctx.sender();
        // Create member card for the package publisher, so that they can invite
        // new members 
        let first_member_membership_card = MembershipCard {
            id: object::new(ctx)
        };
        transfer::transfer(first_member_membership_card, publisher);
    }

    // Function that lets anyone who is already a member (i.e. has a
    // MembershipCard) invite a new member (i.e. create a MembershipCard for 
    // them)
    fun invite_member(_: &MembershipCard, new_member: address, ctx: &mut TxContext) {
        // Create a MembershipCard for a new_member
        let new_member_membership_card = MembershipCard {
            id: object::new(ctx)
        };
        transfer::transfer(new_member_membership_card, new_member);
    }

    // Some functionality that requires a club membership to access
    fun restricted_club_functionality(_: &MembershipCard) {...}
}
```

This pattern can also be modified to restrict the total number of people that a member can invite. This could be done, for example, by keeping a field `members_invited` on the `MembershipCard` and incrementing this value by 1 every time they call `invite_member`, and then adding a check to `invite_member` that `members_invited` is not higher than some maximum number.

# Key Pattern (NFT Access)

Another method for restricting access is to require users to pay for access to a function. You can do this by offering a "Key" object for sale in the smart contracts, which, after it has been bought can be used to access the smart contract functionality. In code that would look like this:

```rust
module ticket_package::key_module {
    use iota::iota::IOTA;

    public struct Key has key {
        id: UID
    }

    public fun buy_key(payment: Coin<IOTA>, ctx: &mut TxContext) {
        ... // Process payment, keep Coin in smart contract or send to admin

        let key = Key {
            id: object::new(ctx)
        };
        transfer::transfer(key, ctx.sender());
    }

    // Function that requires a Key to execute
    public fun use_key_for_restricted_function(key: &Key, ...) {
        // Do something
        ...
    }
}
```

# One-Time Key Pattern (or Ticket Pattern)

A variation on the Key Pattern is the One-Time Key Pattern, or the Ticket Pattern. In this pattern, you require users to buy a ticket to use a function. The ticket is destroyed immediately when it is used to call that function. In that sense the ticket functions as a one-time key.

The code for this is very similar to the code for the Key Pattern; major difference being that the `Ticket` is destroyed immediately after the function is called. 

```rust
module ticket_package::ticket_module {
    use iota::iota::IOTA;

    public struct Ticket has key {
        id: UID
    }

    public fun buy_ticket(payment: Coin<IOTA>, ctx: &mut TxContext) {
        ... // Process payment, keep Coin in smart contract or send to admin

        let ticket = Ticket {
            id: object::new(ctx)
        };
        transfer::transfer(ticket, ctx.sender());
    }

    // Function that requires a ticket to execute
    public fun use_ticket_for_restricted_function(ticket: &Ticket, ...) {
        // Do something
        ...

        // Destroy ticket after use
        let Ticket { id } = ticket;
        object::delete(id);
    }
}
```

Note that we destroy the ticket by unwrapping the `Ticket` struct and calling `object::delete` on it. This guarantees that the ticket can only be used once.

# No Restrictions

The most permissive access pattern is the one with no restrictions: Everyone can mutate an object. In code that looks like this:

```rust
module no_restrictions_package::no_restrictions_module {
    public struct SharedMutableObject has key {
        id: UID,
        counter: u64
    }

    // Create a `SharedMutableObject` when module is published
    fun init(ctx: &mut TxContext) {
        let shared_object = SharedMutableObject {
            id: object::new(ctx),
            counter: 0
        };

        transfer::share_object(shared_object);
    }

    // Anyone can call this
    fun increment_counter(shared_object: &mut SharedMutableObject) {
        shared_object.counter = shared_object.counter + 1;
    }
    
    // Anyone can call this. Will error if `shared_object.count == 0`
    fun decrement_counter(shared_object: &mut SharedMutableObject) {
        shared_object.counter = shared_object.counter - 1;
    }
}
```

Note that objects can only be created in the module that defines them. With the code above, this means that the only time a `SharedMutableObject` is ever created is during module initialization. (Though, of course, you can define a function that creates them.)

After publishing the module above, anyone can use the address of the shared `SharedMutableObject` and call `increment_counter` or `decrement_counter` on it.

As a last note, be aware that you can only access and mutate the `count` field of `SharedMutableObject`s inside the `no_restrictions_module`. Any other module that wants to modify the state of a `SharedMutableObject` can only do so through functions define inside the `no_restrictions_module`.

# Conclusion

That's all! Thanks for reading. If you'd like to learn more about Move, check out my other articles [here](../).
