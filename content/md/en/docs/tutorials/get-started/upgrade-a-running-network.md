---
title: Upgrade a running network
description: Illustrates ways you can update a running node.
keywords:
  - forkless upgrade
  - runtime upgrade
---

Unlike many blockchains, the Substrate development framework supports **forkless upgrades** to the runtime that is the core of the blockchain.
Most blockchain projects require a [hard fork](/reference/glossary/#fork) of the code base to support ongoing development of new features or enhancements to existing features.
With Substrate, you can deploy enhanced runtime capabilities—including breaking changes—without a hard fork.
Because the definition of the runtime is itself an element in the state of a Substrate-based chain, network participants can update this value by calling the [`set_code`](https://paritytech.github.io/substrate/master/frame_system/pallet/enum.Call.html#variant.set_code) function in a transaction.
Because updates to the runtime state are validated using the blockchain's consensus mechanisms and cryptographic guarantees, network participants can use the blockchain itself to distribute updated or extended runtime logic without needing to fork the chain or release a new blockchain client.

This tutorial illustrates how to upgrade the runtime without creating a fork of the code base or stopping the progress of the chain.
In this tutorial, you'll make the following changes to a Substrate runtime on a running network node:

- Add the Scheduler pallet to the runtime.
- Submit a transaction to upload the modified runtime onto a running node.
- Use the Scheduler pallet to increase the minimum balance for network accounts.

## Before you begin

Before you begin, verify the following:

- You have configured your environment for Substrate development by installing [Rust and the Rust toolchain](/install/).

- You have completed [Build a local blockchain](/tutorials/get-started/build-local-blockchain/) and have the Substrate node template installed locally.

* You have reviewed [Add a pallet to the runtime](/tutorials/work-with-pallets/add-a-pallet) for an introduction to adding a new pallet to the runtime.

## Tutorial objectives

By completing this tutorial, you will accomplish the following objectives:

- Use the Sudo pallet to simulate governance for a chain upgrade.

- Upgrade the runtime for a running node to include a new pallet.

- Submit a transaction to upload the modified runtime onto a running node.

- Use the Scheduler pallet to schedule an upgrade for a runtime.

## Authorize an upgrade using the Sudo pallet

Typically, runtime upgrades are managed through governance with community members voting to approve or reject upgrade proposals.
TInplace of governance, tis tutorial uses the Sudo pallet and the `Root` origin to identify the runtime administrator with permission to upgrade the runtime.
Only this root-level administrator can update the runtime by calling the `set_code` function.
The Sudo pallet enables you to invoke the `set_code` function using the `Root` origin by specifying the account that has root-level administrative permissions.

By default, the chain specification file for the node template specifies that the `alice` development account is the owner of the Sudo administrative account.
Therefore, this tutorial uses the `alice` account to perform runtime upgrades.

### Resource accounting for runtime upgrades

Function calls that are dispatched to the Substrate runtime are always associated with a [weight](/reference/glossary/#weight) to account for resource usage.
The FRAME System module sets boundaries on the block length and block weight that these transactions can use.
However, the `set_code` function is intentionally designed to consume the maximum weight that can fit in a block.
Forcing a runtime upgrade to consume an entire block prevents transactions in the same block from executing on different versions of a runtime.

The weight annotation for the `set_code` function also specifies that the function is in the `Operational` class because it provides network capabilities.
Function calls that are identified as operational:

- Can consume the entire weight limit of a block.
- Are given maximum priority.
- Are exempt from paying transaction fees.

### Managing resource accounting

In this tutorial, the [`sudo_unchecked_weight`](https://paritytech.github.io/substrate/master/pallet_sudo/pallet/enum.Call.html#variant.sudo_unchecked_weight) function is used to invoke the `set_code` function for the runtime upgrade.
The `sudo_unchecked_weight` function is the same as the `sudo` function except that it supports an additional parameter to specify the weight to use for the call.
This parameter enables you to work around resource accounting safeguards to specify a weight of zero for the call that dispatches the `set_code` function.
This setting allows for a block to take _an indefinite time to compute_ to ensure that the runtime upgrade does not fail, no matter how complex the operation is.
It can take all the time it needs to succeed or fail.

## Add the Scheduler pallet to the runtime

By default, the node template doesn't include the [Scheduler pallet](https://paritytech.github.io/substrate/master/pallet_scheduler/index.html) in its runtime.
To illustrate a runtime upgrade, you can add the Scheduler pallet to a running node.

### Start the local node

To upgrade the runtime:

1. Open a terminal shell on your computer.

1. Change to the root directory where you compiled the Substrate node template.
   
1. Start the previously-compiled local node in development mode by running the following command:

   ```bash
   cargo run --release -- --dev
   ```

   Leave this node running.
   You can edit and re-compile to upgrade the runtime without stopping or restarting the running node.

### Add Scheduler to the runtime dependencies

To update the dependencies for the runtime to include the Scheduler pallet:

1. Open a second terminal shell window or tab.

1. Change to the root directory where you compiled the Substrate node template.

2. Open the `runtime/Cargo.toml` file in a text editor.

3. Add the Scheduler pallet as a dependency.

   ```toml
   [dependencies]
   ...
   pallet-scheduler = { version = "4.0.0-dev", default-features = false, git = "https://github.com/paritytech/substrate.git", branch = "polkadot-v0.9.28" }
   ...
   ```

   Be sure to use the same version and branch information for the Scheduler pallet as you see used for the other pallets included in the runtime.
   In this example, all of the pallets in the node template runtime use `version = "4.0.0-dev"` and `branch = "polkadot-v0.9.28"`.

4. Add the Scheduler pallet to the `features` list.

   ```toml
   [features]
   default = ["std"]
   std = [
     ...
     "pallet-scheduler/std",
     ...
   ```

5. Save your changes and close the `Cargo.toml` file.

### Add the Scheduler to the runtime

To add the Scheduler types and configuration trait:

1. Open the `runtime/src/lib.rs` file in a text editor.

2. Add the following trait dependency near the top of the file:

   ```rust
   pub use frame_support::traits::EqualPrivilegeOnly;
   ```

3. Add the types required by the Scheduler pallet.

   ```rust
   parameter_types! {
     pub MaximumSchedulerWeight: Weight = 10_000_000;
     pub const MaxScheduledPerBlock: u32 = 50;
   }
   ```

4.  Add the implementation for the Config trait for the Scheduler pallet.

   ```rust
   impl pallet_scheduler::Config for Runtime {
     type Event = Event;
     type Origin = Origin;
     type PalletsOrigin = OriginCaller;
     type Call = Call;
     type MaximumWeight = MaximumSchedulerWeight;
     type ScheduleOrigin = frame_system::EnsureRoot<AccountId>;
     type MaxScheduledPerBlock = MaxScheduledPerBlock;
     type WeightInfo = ();
     type OriginPrivilegeCmp = EqualPrivilegeOnly;
     type PreimageProvider = ();
     type NoPreimagePostponement = ();
   }
   ```

1. Add the Scheduler pallet inside the `construct_runtime!` macro.

   ```rust
   construct_runtime!(
     pub struct Runtime 
     where
        Block = Block,
        NodeBlock = opaque::Block,
        UncheckedExtrinsic = UncheckedExtrinsic
     {
        /*** snip ***/
        Scheduler: pallet_scheduler,
     }
   );
   ```

1. Increment the [`spec_version`](https://paritytech.github.io/substrate/master/sp_version/struct.RuntimeVersion.html#structfield.spec_version) in the `RuntimeVersion` struct](https://paritytech.github.io/substrate/master/sp_version/struct.RuntimeVersion.html) to upgrade runtime version.

   ```rust
   pub const VERSION: RuntimeVersion = RuntimeVersion {
     spec_name: create_runtime_str!("node-template"),
     impl_name: create_runtime_str!("node-template"),
     authoring_version: 1,
     spec_version: 101,  // *Increment* this value, the template uses 100 as a base
     impl_version: 1,
     apis: RUNTIME_API_VERSIONS,
     transaction_version: 1,
   };
   ```

   Review the components of the `RuntimeVersion` struct:

   - `spec_name` specifies the name of the runtime.
   - `impl_name` specifies the name of the outer node client.
   - `authoring_version` specifies the version for [block authors](/reference/glossary#author).
   - `spec_version` specifies the version of the runtime.
   - `impl_version` specifies the version of the outer node client.
   - `apis` specifies the list of supported APIs.
   - `transaction_version` specifies the version of the [dispatchable function](/reference/glossary#dispatch) interface.

   To upgrade the runtime, you must _increase_ the `spec_version`.
   For more information, see the [FRAME System](https://github.com/paritytech/substrate/tree/master/frame/system) module and [`can_set_code`](https://github.com/paritytech/substrate/blob/master/frame/system/src/lib.rs#L1566) function.

2.  Save your changes and close the `runtime/src/lib.rs` file.

### Recompile and connect to the local node

1. Verify that the local node continues to run in the first terminal.
   
2. In the second terminal where you updated the runtime `Cargo.toml` and `lib.rs` files, recompile the runtime by running the following command

   ```shell
   cargo build --release -p node-template-runtime
   ```

   The `--release` command-line option requires a longer compile time.
   However, it generates a smaller build artifact that is better suited for submitting to the blockchain network.
   Storage optimization is _critical_ for any blockchain.
   With this command, the build artifacts are output to the `target/release` directory.
   The WebAssembly build artifacts are in the `target/release/wbuild/node-template-runtime` directory.
   For example, you should see the following WebAssembly artifacts:

   ```text
   node_template_runtime.compact.compressed.wasm
   node_template_runtime.compact.wasm
   node_template_runtime.wasm
   ```

## Submit a transaction to upgrade the runtime

You now have a WebAssembly artifact that describes the modified runtime logic.
However, the running node isn't using the upgraded runtime yet.
To complete the upgrade, you need to submit a transaction that updates the node to use the upgraded runtime.

To update the network with the upgraded runtime:

1. Open the [Polkadot/Substrate Portal](https://polkadot.js.org/apps/#/explorer) in a browser and connect to the local node.

1. Click **Developer** and select **Extrinsics** to submit a transaction for the runtime to use the new build artifact.

2. Select the administrative  **Alice** account.

3. Select the **sudo** pallet and the **sudoUncheckedWeight(call, weight)** function.
   
4. Select **system** and **setCode(code)** as the call to make using the Alice account.

5. Click **file upload**, then select or drag and drop the compact and compressed WebAssembly file—`node_template_runtime.compact.compressed.wasm`—that you generated for the runtime.

   For example, navigate to the `target/release/wbuild/node-template-runtime` directlroy and select `node_template_runtime.compact.compressed.wasm` as the file to upload.

6. Leave the **weight** parameter with the default of `0`.

   ![Runtime upgrade settings](/media/images/docs/tutorials/forkless-upgrade/set-code-transaction.png)

7.  Click **Submit Transaction**.

8.  Review the authorization, then click **Sign and Submit**.

9. Click **Network** and select **Explorer** to see that there has been a successful sudo.Sudid event.
   
   ![Successful sudo event](/media/images/docs/tutorials/forkless-upgrade/set-code-sudo-event.png)   

   After the transaction is included in a block, the node template version number indicates that the runtime version is now `101`.
   For example:

   ![Updated runtime version is 101](/media/images/docs/tutorials/forkless-upgrade/runtime-version-101.png)

   If your local node is producing blocks in the terminal that match what is displayed in the browser, you have completed a successful runtime upgrade.

## Schedule an upgrade

In the previous upgrade example, you used the `sudo_unchecked_weight` function to skip the accounting safeguards that limit block length and weight to allow the `set_code` function call to take as long as necessary to complete the runtime upgrade.
Now that you have updated the node template to include the Scheduler pallet, however, you can perform a **scheduled** runtime upgrade. 
A scheduled runtime upgrade ensures that the `set_code` function call is the only transaction included in a block.

### Prepare an upgraded runtime

In the previous upgrade example, you added a whole new pallet to the runtime.
This upgrade example is more straightforward and only requires updating values that already exist
in the `runtime/src/lib.rs` file.
For this upgrade example, you are going to increase the minimum balance required for network accounts.
This minimum balance—called an [existential deposit](/reference/glossary#existential-deposit)—is a constant in the runtime and has a value of 500 in the node template by default.

To modify the value of the existential deposit for a runtime upgrade:

1. Verify that the local node continues to run in the first terminal.

1. Open a second terminal shell window or tab.

1. Change to the root directory where you compiled the Substrate node template.

3. Open the `runtime/src/lib.rs` file in a text editor.

1. Update the `spec_version` for the runtime to 102.
   
   ```rust
   pub const VERSION: RuntimeVersion = RuntimeVersion {
      spec_name: create_runtime_str!("node-template"),
      impl_name: create_runtime_str!("node-template"),
      authoring_version: 1,
      spec_version: 102,  // Increment this value.
      impl_version: 1,
      apis: RUNTIME_API_VERSIONS,
      transaction_version: 1,
   };

1. Update the value for the EXISTENTIAL_DEPOSIT for the Balances pallet.
   
   ```rust
   pub const EXISTENTIAL_DEPOSIT: u128 = 1000 // Update this value.
   ```
   
   This change increases the minimum balance an account is required to have on deposit to be viewed as a valid active account.
   This change doesn't remove any accounts with balances between 500 and 1000.
   Removing accounts would require a storage migration.
   For information about upgrading data storage, see [storage migration](/build/upgrade-the-runtime/#storage-migrations)

1. Save your changes and close the `runtime/src/lib.rs` file.

1. Build the upgraded runtime by running the following command:
   
   ```bash
   cargo build --release -p node-template-runtime
   ```
   
   This command generates a new set of build artifacts. overwriting the previous set of build artifacts.
   If you want to save the previous set of artifacts, copy them to another location before compiling the node template.

1. Verify the WebAssembly build artifacts are in the `target/release/wbuild/node-template-runtime` directory.

### Submit a transaction to schedule the upgrade

You now have a WebAssembly artifact that describes the modified runtime logic.
The Scheduler pallet is configured with the `Root` origin as its [`ScheduleOrigin`](https://paritytech.github.io/substrate/master/pallet_scheduler/pallet/trait.Config.html#associatedtype.ScheduleOrigin).
With this configuration, you can use the `sudo` function—not_ `sudo_unchecked_weight`—to invoke the `schedule` function.

To schedule the runtime upgrade:

1. Open the [Polkadot/Substrate Portal](https://polkadot.js.org/apps/#/explorer) in a browser and connect to the local node.

2. Click **Developer** and select **Sudo**.

3. Select **scheduler** and **schedule(when, maybePeriodic, priority, call)**.

   - Use the default values for the **maybePeriodic** (empty) and **priority** (0) parameters.
   - Select **system** and **setCode(code)** as the call.
   - Click **file upload** and select the compressed WebAssembly file you generated for the runtime.

1. Check the current block number and set the **when** parameter to a block number 10 to 20 blocks from the current block, then click **Submit Sudo**.
   
   ![Settings to schedule a runtime upgrade](/media/images/docs/tutorials/forkless-upgrade/sudo-schedule-upgrade.png)

2. Review the authorization and click **Sign and Submit**.

1. Monitor block production in the terminal or the [Network Explorer](https://polkadot.js.org/apps/#/explorer?rpc=ws://127.0.0.1:9944) to watch as this scheduled call takes place.
   
   After the target block has been included in the chain, the node template version number indicates that the runtime version is now `102`.

2. Verify the constant value by querying the chain state in the the [Polkadot/Substrate Portal](https://polkadot.js.org/apps/#/chainstate/constants?rpc=ws://127.0.0.1:9944).
   
   - Select the **balances** pallet.
   - Select **existentialDeposit** as the constant value as the value to query.

   ![Verify the constant value change](/media/images/docs/tutorials/forkless-upgrade/constant-value-lookup.png)
   
## Where to go next

- [Storage migrations](/build/upgrade-the-runtime/#storage-migration)
  <!-- TODO NAV.YAML -->
  <!-- add  back ABOVE -->
  <!-- - [How-to: Storage migration](/reference/how-to-guides/basics/storage-migration/) -->
