---
title:  picoCTF 2022 - Binary Exploitation - Solfire (500 points) - Part I
description: We start off by reversing the Solana eBPF binary by jitting it to x86_64 bytecode an examining its functions.
hide_index: true
image:
  path: /img/reference/pico-ctf-2022-00000.png
  width: 1024
  height: 512
  thumb: /img/reference/pico-ctf-2022-00000.thumb.png
  alt: Linux Terminal
---

# picoCTF 2022 - Solfire - Part I

> **Note**: This article is part of our [picoCTF 2022 Greatest Hits Guide]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits.md %}).

Solfire had the fewest solves of all the challenges in picoCTF 2022. Only a handful of teams were able to solve it during the competition. We came to the challenge late, having done all the other challenges first. Our approach isn't the most elegant, but it did (eventually) get the job done.

This is the first part of a *three part series*. In Part I, we cover reversing the eBPF binary we were given.

I. [Part I - Reversing the Binary]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits/solfire.md %}) (you are here)  
II. [Part II - Environment Setup]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits/solfire/part-2.md %})  
III. Part III - Exploitation (*coming soon*)

## The Problem

For this challenge we get thrown straight into the deep end. We get access to a Dockerfile, a `solfire.so` binary, and some rust source code. We are also given the following cryptic hint:

> What is debt? A perversion of a promise?  
Surely one has to [pay one's debts](https://blog.osec.io/tutorials/2022-03-14-solana-security-intro/).

Welp. We don't really know rust. This binary they've given us isn't an x86_64 binary, it's for an architecture called `eBPF`. Looks like we've got our work cut out for us.

## Decoding the Rust

The first couple lines of the rust source code tells us some basic information about this challenge. First up, this is clearly a `Solana` based cryptocurrency/blockchain challenge.

```rust
use solana_sdk::{
    instruction::{AccountMeta, Instruction},
    transaction::Transaction,
};

use solana_program::{
    pubkey::Pubkey
};
use anchor_client::solana_sdk::system_instruction::transfer;
use poc_framework::{
    solana_sdk::{self, signature::Keypair, signer::Signer},
    Environment, LocalEnvironment,
};
```

To be honest, Solana wasn't something we were familiar with, but we did recently complete the excellent [Damn Vulnerable DeFi](https://www.damnvulnerabledefi.xyz/) Ethereum challenges, so maybe some of that knowledge will pay-off here.

Researching the packages involved, we learn that [poc_framework](https://github.com/neodyme-labs/solana-poc-framework) is something like Hardhat - a way to deploy a test environment with our own users and smart contracts.

```rust
let mut env_builder = LocalEnvironment::builder();
let mut env = env_builder.build();

let program_pubkey = env.deploy_program("./solfire.so");
let solve_pubkey = env.deploy_program(solve_file.path());

let user = Keypair::new();

writeln!(socket, "program pubkey: {}", program_pubkey)?;
writeln!(socket, "solve pubkey: {}", solve_pubkey)?;
writeln!(socket, "user pubkey: {}", user.pubkey())?;

let (vault, _) = Pubkey::find_program_address(&["vault".as_ref()], &program_pubkey);
```

Looks like this code deploys two smart contracts: the `solfire.so` binary and **one of our own**. There is also a `user` account. We're given the public keys for both of the contracts and the user. There is also a *vault* account, which is somehow derived from the smart-contract's public key, but we aren't given any information about it.

```rust
const TARGET_AMT: u64 = 50_000;
const INIT_BAL: u64 = 10;
const VAULT_BAL: u64 = 1_000_000;
env.execute_as_transaction(
    &[transfer(
        &env.payer().pubkey(),
        &user.pubkey(),
        INIT_BAL,
    ),
    transfer(
        &env.payer().pubkey(),
        &vault,
        VAULT_BAL,
    )
    ],
    &[&env.payer()],
);
```

Here we see the user ends up with 10 lamports (`INIT_BAL`) and the vault ends up with 1,000,000 lamports (`VAULT_BAL`). We can surmise that the `solfire.so` contract is in control of the vault.

Following that code there is a section on parsing some account metadata stuff - to be honest we don't really understand that part yet. This is new to us and must be specific to Solana - *it doesn't resemble anything we've seen while doing the Ethereum challenges*.

Finally there's this bit, which is where we learn how to get the flag:

```rust
line.clear();
assert!(reader.read_line(&mut line)? != 0);
let ix_data_len: usize = line.trim().parse()?;
let mut ix_data = vec![0; ix_data_len];

reader.read_exact(&mut ix_data)?;

let ix = Instruction::new_with_bytes(
    solve_pubkey,
    &ix_data,
    metas
);

let tx = Transaction::new_signed_with_payer(
    &[ix],
    Some(&user.pubkey()),
    &vec![&user],
    env.get_recent_blockhash(),
);

env.execute_transaction(tx);
let user_bal = env.get_account(user.pubkey()).unwrap().lamports;
writeln!(socket, "user bal: {:?}", user_bal)?;
writeln!(socket, "vault bal: {:?}", env.get_account(vault).unwrap().lamports)?;

if user_bal > TARGET_AMT {
    writeln!(socket, "congrats!")?;
    if let Ok(flag) = env::var("FLAG") {
        writeln!(socket, "flag: {:?}", flag)?;
    } else {
        writeln!(socket, "flag not found, please contact admin")?;
    }
}
```

Aha! We get to specify some instruction data and call a single instruction on our smart contract. If, after the transaction executes, the user's account balance is > 50,000 lamports then the flag will be printed.

**All we have to do is turn our measly 10 lamports into over 50,000 lamports in a single transaction! _How hard could that be_...**

## Decoding the eBPF Smart Contract

The next problem is decoding this `solfire.so` binary. Apparently there are some [ghidra plugins](https://github.com/Toizi/eBPF-for-Ghidra) for that, but getting those to work would probably mean compiling some java. Since compiling java and dealing with setting up the environment and plugins correctly is my least favourite thing in the world - let's instead see if we can crack this nut in a **more difficult** and **time consuming** way.

In general `readelf` does seem to understand the binary. It's a dynamic library, so let's see what functions it exports:

```
$ readelf --dyn-syms solfire.so

Symbol table '.dynsym' contains 19 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND sol_panic_
     2: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND sol_log_
     3: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND sol_log_pubkey
     4: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND sol_invoke_signed_c
     5: 00000000000000e8   376 FUNC    GLOBAL DEFAULT    1 itoa
     6: 0000000000000878    16 FUNC    GLOBAL DEFAULT    1 log_pubkey
     7: 0000000000000bd8   248 FUNC    GLOBAL DEFAULT    1 is_system_program
     8: 0000000000001758  1384 FUNC    GLOBAL DEFAULT    1 handle_withdraw
     9: 0000000000000888   160 FUNC    GLOBAL DEFAULT    1 strncmp
    10: 0000000000000260   528 FUNC    GLOBAL DEFAULT    1 log_int
    11: 0000000000000a78   352 FUNC    GLOBAL DEFAULT    1 str_contains
    12: 00000000000011d8   208 FUNC    GLOBAL DEFAULT    1 check_owner
    13: 00000000000012a8  1200 FUNC    GLOBAL DEFAULT    1 handle_deposit
    14: 0000000000000928   336 FUNC    GLOBAL DEFAULT    1 strcmp
    15: 0000000000000470  1032 FUNC    GLOBAL DEFAULT    1 b58enc
    16: 0000000000000cd0  1288 FUNC    GLOBAL DEFAULT    1 handle_create
    17: 0000000000001cc0   760 FUNC    GLOBAL DEFAULT    1 solfire
    18: 0000000000001fb8   872 FUNC    GLOBAL DEFAULT    1 entrypoint
```
{:.contains-term}

After a bit of research we decide that we're probably most interested in the `entrypoint` function, and then the `solfire` function, and finally the 3 functions with a similar name: `handle_create`, `handle_withdraw` and `handle_deposit`.

Unfortunately, my version of the regular `objdump` binary doesn't seem to like this architecture. However, it looks like llvm-12 has a version of objdump that can read and disassembly eBPF binaries.

```
$ docker run --rm -it -v $PWD:/work -w /work silkeh/clang:12 llvm-objdump -S solfire.so

solfire.so:     file format elf64-bpf


Disassembly of section .text:

00000000000000e8 <itoa>:
      29:       bf 36 00 00 00 00 00 00 r6 = r3
      30:       bf 27 00 00 00 00 00 00 r7 = r2
      31:       bf 18 00 00 00 00 00 00 r8 = r1
      32:       b7 01 00 00 11 00 00 00 r1 = 17
      33:       2d 61 06 00 00 00 00 00 if r1 > r6 goto +6 <LBB0_2>
      34:       18 01 00 00 b1 3d 00 00 00 00 00 00 00 00 00 00 r1 = 15793 ll
      36:       b7 02 00 00 1f 00 00 00 r2 = 31
      37:       b7 03 00 00 05 00 00 00 r3 = 5
      38:       b7 04 00 00 00 00 00 00 r4 = 0
      39:       85 10 00 00 ff ff ff ff call -1

0000000000000140 <LBB0_2>:
      40:       b7 01 00 00 01 00 00 00 r1 = 1
      41:       15 08 06 00 00 00 00 00 if r8 == 0 goto +6 <LBB0_5>
      42:       b7 01 00 00 00 00 00 00 r1 = 0
      43:       bf 82 00 00 00 00 00 00 r2 = r8

0000000000000160 <LBB0_4>:
      44:       bf 23 00 00 00 00 00 00 r3 = r2
      45:       07 01 00 00 01 00 00 00 r1 += 1
      46:       3f 62 00 00 00 00 00 00 r2 /= r6
      47:       3d 63 fc ff 00 00 00 00 if r3 >= r6 goto -4 <LBB0_4>

0000000000000180 <LBB0_5>:
      48:       bf 12 00 00 00 00 00 00 r2 = r1
      49:       67 02 00 00 20 00 00 00 r2 <<= 32
      50:       77 02 00 00 20 00 00 00 r2 >>= 32
      51:       bf 74 00 00 00 00 00 00 r4 = r7
      52:       0f 24 00 00 00 00 00 00 r4 += r2
      53:       b7 03 00 00 00 00 00 00 r3 = 0
      54:       73 34 00 00 00 00 00 00 *(u8 *)(r4 + 0) = r3
      55:       07 01 00 00 ff ff ff ff r1 += -1
      56:       bf 84 00 00 00 00 00 00 r4 = r8

...
```
{:.contains-term}

We stare at the assembly for a while. *We stare some more*. We give up.

We're going to need some tooling support. The generated code is extremely memory-heavy, with lots of pointers and offsets.

At some point I stumbled upon [uBPF](https://github.com/iovisor/ubpf) which can jit eBPF bytecode to x86_64 instructions in userland. For better or for worse, I decided to take the following approach (which is admittedly a bit *crazy*):

1. Patch uBPF to dump jitted bytecode, and to basically ignore function calls by always emitting a call to the address `0xdeadbeef`.
2. Extract the `.text` section of the `solfire.so` binary:  
  `llvm-objcopy solfire.so --dump-section .text=solfire.text.bin`
3. For each of the functions of interest (`entrypoint`, `solfire`, `handle_create`, `handle_withdraw`, and `handle_deposit`) extract the bytecode from `solfire.text.bin` and run it through uBPF, resulting in some x86_64 bytecode.
4. Manually disassemble the x86_64 bytecode and generate the equivalent x86_64 assembly for that routine, correcting the function calls as they were all broken.
5. Extract the `.rodata` section, and correct the assembly such that it properly references the static strings contained there.
6. Compile my own object file with stubs for the missing external functions. Also, define some of the basic structures from emsdk and compile with debugging symbols enabled so that they are available for use by the decompiler.

This was obviously a highly-manual process. I've since improved it somewhat by allowing uBPF to directly intake the eBPF `.so` and output a static `.o` file in x86_64, *but that's a post for another time*. In any case, this is what I did during the competition.

**The result is this [binary]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits/solfire/solfire_x86_64 %}), which is a _normal_ x86_64 binary that is compatible with all of the regular tools (ie: ghidra).**

We can now analyze each of the functions of interest in turn:

### entrypoint

I didn't spend a lot of time on this function. The normal behavior is to use the library deserializer and then just call another function, in this case `solfire`.

### solfire

<div class="panel-group">
  <div class="panel panel-info">
    <div class="panel-heading">
      <h3 class="panel-title">
        <a data-toggle="collapse" href="#collapseCode" aria-expanded="false" aria-controls="collapseCode">
        Ghidra output for solfire <small>(click to expand)</small>
        <i class="glyphicon glyphicon-menu-left pull-right"></i>
        <i class="glyphicon glyphicon-menu-down pull-right"></i>
        </a>
      </h3>
    </div>
    <div id="collapseCode" class="panel-collapse collapse">
<div class="panel-body" markdown="1">

```c
undefined8 solfire(SolParameters *param_1)
{
  bool bVar1;
  long lVar2;
  uint line;
  uint extraout_EDX;
  ulong uVar3;
  char *str;
  ulong uVar4;
  byte *b58;
  ulong uVar5;
  uint64_t buf_size;
  byte buf [100];
  SolAccountInfo *account_0;
  int cmd;
  
  account_0 = param_1->ka;
  buf_size = 100;
  b58 = buf;
  b58enc((char *)b58,&buf_size,account_0->key,0x20);
  uVar3 = 0;
  line = (uint)buf[0];
  while( true ) {
    uVar4 = 0;
    if (buf[0] != 0) {
      uVar4 = 0;
      do {
        lVar2 = uVar4 + 1;
        uVar4 = uVar4 + 1;
      } while (buf[lVar2] != 0);
    }
    if (uVar4 <= uVar3) break;
    uVar5 = 0;
    while (b58[uVar5] == s_C1ock_00013d50[uVar5]) {
      if ((b58[uVar5] == 0) || (bVar1 = 3 < uVar5, uVar5 = uVar5 + 1, bVar1)) {
        if (uVar4 <= uVar3) goto lbl6011c;
        if (*(ulong *)(account_0->data + 0x20) < 0x6230b800) {
          if (param_1->data_len < 4) {
            sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0xc5);
            line = extraout_EDX;
          }
          cmd = *(int *)param_1->data;
          if (cmd == 0) {
            handle_create(param_1);
          }
          else {
            if (cmd == 1) {
              handle_deposit(param_1);
            }
            else {
              if (cmd != 2) {
                sol_panic(s_invalid_op_choice_00013d64,0x11,line);
                log_int(*(int *)param_1->data);
                return 0x200000000;
              }
              handle_withdraw(param_1);
            }
          }
          return 0;
        }
        str = s_it_is_too_late_to_do_this_challe_00013de0;
        cmd = 0x26;
        goto lbl60155;
      }
    }
    b58 = b58 + 1;
    uVar3 = uVar3 + 1;
  }
lbl6011c:
  str = s_bad_C1ock_account_00013d9f;
  cmd = 0x11;
lbl60155:
  sol_log(str,cmd);
  return 0x200000000;
}
```

</div>
    </div>
  </div>
</div>

This function is a little long, but as the functional "entrypoint" for this smart-contract it does the following *every time* any of the other functions (handle_create, handle_deposit, handle_withdraw) are invoked:

1. b58 encode the public key for `accounts[0]` (**_NOTE_: this is probably a lot of computation**)
2. Search the encoded address for the sub-string `"C1ock"`
3. If `"C1ock"` was **not** found, log `"bad C1ock account"` and quit
4. If `"C1ock"` **was** found, read 8 bytes from that account's data, at offset `0x20`, and compare that value to `0x6230b800`
5. If the value is greater than (or equal to) `0x6230b800` log the message `"it is too late to do this challenge :("` and quit
6. If the value is less than `0x6230b800`, ensure that we were given at least 4 bytes of instruction data
7. Treat the first 4 bytes of instruction data as an integer and do the following:  
  a) If the value is `0`, call `handle_create`  
  b) If the value is `1`, call `handle_deposit`  
  c) If the value is `2`, call `handle_withdraw`  
  d) Otherwise, log the value and quit

Basically, `account[0]` **_must_** have an address that contains `"C1ock"`. It turns out there is a built-in Solana [SysVar](https://docs.solana.com/developing/runtime-facilities/sysvars#clock) that has an address that contains the string `"C1ock"`. But what is at offset `0x20`? A unix timestamp. And what time is `0x6230b800`? **Tue Mar 15 2022 16:00:00 GMT+0000** - ie: the exact start of picoCTF 2022. Obviously we somehow need to use something else here, or maybe somehow invent time travel.

### handle_create

<div class="panel-group">
  <div class="panel panel-info">
    <div class="panel-heading">
      <h3 class="panel-title">
        <a data-toggle="collapse" href="#collapseCode2" aria-expanded="false" aria-controls="collapseCode2">
        Ghidra output for handle_create <small>(click to expand)</small>
        <i class="glyphicon glyphicon-menu-left pull-right"></i>
        <i class="glyphicon glyphicon-menu-down pull-right"></i>
        </a>
      </h3>
    </div>
    <div id="collapseCode2" class="panel-collapse collapse">
<div class="panel-body" markdown="1">

```c
undefined8 handle_create(SolParameters *param_1)
{
  uint8_t uVar1;
  bool bVar2;
  ulong uVar3;
  uint8_t *puVar4;
  SolInstruction inst;
  uint8_t inst_buf [52];
  SolAccountMeta meta [2];
  SolSignerSeeds seeds;
  SolSignerSeed seed;
  uint8_t inst_data_offset_4;
  SolAccountInfo *accounts;
  SolPubkey *local_pubkey;
  
  sol_log(s_handle_create_00013e42,0xd);
  if (param_1->ka_num != 5) {
    sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x22);
  }
  accounts = param_1->ka;
  inst_data_offset_4 = param_1->data[4];
  inst_buf[24] = '\0';
  inst_buf[25] = '\0';
  inst_buf[26] = '\0';
  inst_buf[27] = '\0';
  inst_buf[28] = '\0';
  inst_buf[29] = '\0';
  inst_buf[30] = '\0';
  inst_buf[31] = '\0';
  inst_buf._16_4_ = 0;
  inst_buf[20] = '\0';
  inst_buf[21] = '\0';
  inst_buf[22] = '\0';
  inst_buf[23] = '\0';
  inst_buf._8_4_ = 0;
  inst_buf._12_4_ = 0;
  inst_buf._0_2_ = 0;
  inst_buf._2_2_ = 0;
  inst_buf._4_2_ = 0;
  inst_buf._6_2_ = 0;
  if ((accounts[1].key)->x[0] == '\0') {
    uVar3 = 1;
    do {
      puVar4 = inst_buf + uVar3;
      uVar1 = (accounts[1].key)->x[uVar3];
      if (uVar1 != *puVar4) break;
      bVar2 = uVar3 < 0x1f;
      uVar3 = uVar3 + 1;
    } while (bVar2);
    if (uVar1 == *puVar4) goto lbl30124;
  }
  sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x2a);
lbl30124:
  seed.addr = &inst_data_offset_4;
  seed.len = 1;
  seeds.addr = &seed;
  seeds.len = 1;
  meta[0].pubkey = accounts[4].key;
  meta[0]._8_2_ = 0x101;
  meta[1].pubkey = accounts[2].key;
  meta[1]._8_2_ = 0x101;
  inst_buf._12_4_ = 0x2800;
  inst_buf._16_4_ = 0;
  inst_buf._2_2_ = 0;
  inst_buf._0_2_ = 0;
  inst_buf._4_2_ = 1;
  inst_buf._6_2_ = 0;
  inst_buf._8_4_ = 0;
  local_pubkey = param_1->program_id;
  inst_buf[20] = local_pubkey->x[0];
  inst_buf[21] = local_pubkey->x[1];
  inst_buf[22] = local_pubkey->x[2];
  inst_buf[23] = local_pubkey->x[3];
  inst_buf[24] = local_pubkey->x[4];
  inst_buf[25] = local_pubkey->x[5];
  inst_buf[26] = local_pubkey->x[6];
  inst_buf[27] = local_pubkey->x[7];
  inst_buf[28] = local_pubkey->x[8];
  inst_buf[29] = local_pubkey->x[9];
  inst_buf[30] = local_pubkey->x[10];
  inst_buf[31] = local_pubkey->x[0xb];
  inst_buf[32] = local_pubkey->x[0xc];
  inst_buf[33] = local_pubkey->x[0xd];
  inst_buf[34] = local_pubkey->x[0xe];
  inst_buf[35] = local_pubkey->x[0xf];
  inst_buf._36_8_ = *(undefined8 *)(local_pubkey->x + 0x10);
  inst_buf._44_8_ = *(undefined8 *)(local_pubkey->x + 0x18);
  inst.program_id = accounts[1].key;
  inst.data_len = 0x34;
  inst.data = inst_buf;
  inst.account_len = 2;
  inst.accounts = meta;
  sol_invoke_signed_c(&inst,param_1->ka,(int)param_1->ka_num,&seeds,1);
  return 0;
}
```

</div>
    </div>
  </div>
</div>

This function is a little simpler to understand:

1. Make sure we've passed in exactly 5 accounts
2. Make sure `accounts[1]` key is all zeros. (In b58 encoding, all zeros is represented as `11111111111111111111111111111111` - which is the [system program](https://docs.solana.com/developing/runtime-facilities/programs#system-program))
3. Construct a seed based on 1 byte from the instruction data (at offset 4, which immediately proceeds the 4 bytes used to specify that we wanted to call `handle_create` in the first place).
4. Construct some account metadata, where the first account is `accounts[4]` and the second account is `accounts[2]`, both of which must have `WRITE` and `SIGNER` permissions.
5. Do a cross-program invocation to the [SYSTEM program](https://docs.rs/solana-program/1.10.7/solana_program/system_instruction/enum.SystemInstruction.html). We send `0x34` bytes of instruction data:  
  a) 4 bytes of zeros - Indicates we want the `CreateAccount` instruction  
  b) 8 bytes [`0x01` padded with zeros] - Number of lamports to transfer into the new account  
  c) 8 bytes [`0x2800` padded with zeros] - Space to allocate (in data) for the new account  
  d) 32 bytes [copied from `program_id` ] - Owner of the new account
6. For the `CreateAccount` call, the first account in the metadata (`accounts[4]`) is the *funding* account, and the second account in the metadata (`accounts[2]`) is the created account. Since these accounts must both signed, and it costs 1 lamport, the only choice for `accounts[4]` is user (who starts out with 10 lamports and can sign the transaction).

### handle_deposit

<div class="panel-group">
  <div class="panel panel-info">
    <div class="panel-heading">
      <h3 class="panel-title">
        <a data-toggle="collapse" href="#collapseCode3" aria-expanded="false" aria-controls="collapseCode3">
        Ghidra output for handle_deposit <small>(click to expand)</small>
        <i class="glyphicon glyphicon-menu-left pull-right"></i>
        <i class="glyphicon glyphicon-menu-down pull-right"></i>
        </a>
      </h3>
    </div>
    <div id="collapseCode3" class="panel-collapse collapse">
<div class="panel-body" markdown="1">

```c
undefined8 handle_deposit(SolParameters *param_1)

{
  uint8_t uVar1;
  uint8_t uVar2;
  bool bVar3;
  long lVar4;
  long lVar5;
  ulong uVar6;
  uint64_t *this_ledger;
  SolInstruction instruction;
  uint8_t inst_buf [12];
  SolAccountMeta meta [2];
  SolAccountInfo (*accounts) [5];
  uint32_t *input;
  SolPubkey *account_2_owner;
  SolPubkey *local_pubkey;
  
  sol_log(s_handle_deposit_00013e50,0xe);
  if (param_1->ka_num != 5) {
    sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x50);
  }
  accounts = (SolAccountInfo (*) [5])param_1->ka;
  local_pubkey = (*accounts)[1].key;
  instruction.data = (uint8_t *)0x0;
  instruction.account_len = 0;
  instruction.accounts = (SolAccountMeta *)0x0;
  instruction.program_id = (SolPubkey *)0x0;
  if (local_pubkey->x[0] == '\0') {
    uVar6 = 1;
    do {
      uVar1 = local_pubkey->x[uVar6];
      uVar2 = *(uint8_t *)((long)&instruction.program_id + uVar6);
      if (uVar1 != uVar2) break;
      bVar3 = uVar6 < 0x1f;
      uVar6 = uVar6 + 1;
    } while (bVar3);
    if (uVar1 != uVar2) goto lbl400ed;
  }
  else {
lbl400ed:
    sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x56);
  }
  local_pubkey = param_1->program_id;
  account_2_owner = (*accounts)[2].owner;
  if (account_2_owner->x[0] == local_pubkey->x[0]) {
    uVar6 = 0;
    if (account_2_owner->x[1] == local_pubkey->x[1]) {
      uVar6 = 0;
      do {
        if (uVar6 == 0x1e) goto lbl401cb;
        lVar4 = uVar6 + 2;
        lVar5 = uVar6 + 2;
        uVar6 = uVar6 + 1;
      } while (account_2_owner->x[lVar4] == local_pubkey->x[lVar5]);
    }
    if (0x1e < uVar6) goto lbl401cb;
  }
  sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x44);
lbl401cb:
  if ((*accounts)[3].is_signer == false) {
    sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x5b);
  }
  if (param_1->data_len - 4 < 8) {
    sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x5c);
  }
  input = (uint32_t *)param_1->data;
                    /* input[1] is an ledger "entry" number */
  if (0x280 < input[1]) {
    sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x5f);
  }
                    /* input[2] is the deposit amount */
  if (input[2] == 0) {
    sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x60);
  }
  meta[0].pubkey = (*accounts)[3].key;
  meta[0]._8_2_ = 0x101;
  meta[1].pubkey = (*accounts)[4].key;
  meta[1]._8_2_ = 1;
  inst_buf._2_2_ = 0;
  inst_buf._0_2_ = 2;
  inst_buf._4_6_ = (uint6)input[2];
  inst_buf._10_2_ = 0;
  instruction.program_id = (*accounts)[1].key;
  instruction.data_len = 0xc;
  instruction.data = inst_buf;
  instruction.account_len = 2;
  instruction.accounts = meta;
  sol_invoke_signed_c(&instruction,param_1->ka,(int)param_1->ka_num,
                      (SolSignerSeeds *)&stack0xffffffffffffffd0,0);
  this_ledger = (uint64_t *)((*accounts)[2].data + (ulong)input[1] * 0x10);
  this_ledger[1] = this_ledger[1] + (ulong)input[2];
  return 0;
}
```

</div>
    </div>
  </div>
</div>

This one is a little more complicated than the last, but is still understandable:

1. Make sure we've passed in exactly 5 accounts (still).
2. Make sure `accounts[1]` key is all zeros. (In b58 encoding, all zeros is represented as `11111111111111111111111111111111` - which is the [system program](https://docs.solana.com/developing/runtime-facilities/programs#system-program))
3. Make sure `accounts[2]` has an owner and the public key matches `program_id` (ie: that account should be owned by the current program). **NOTE**: I will refer to `accounts[2]` as the "ledger" account.
4. `accounts[3]` **MUST** have signed the request
5. There should be at least 12 bytes of data
6. Treat the input data as an array of ints. `input[1]` **MUST** be less than or equal to `0x280`. `input[2]` cannot be zero. (Recall: `input[0]` was `1` if handle_deposit was called from `solfire`).
7. Construct some account metadata, where the first account is `accounts[3]` and the second account is `accounts[4]`. `accounts[3]` is a signer, and both accounts must have `WRITE` permissions.
8. Do a cross-program invocation to the [SYSTEM program](https://docs.rs/solana-program/1.10.7/solana_program/system_instruction/enum.SystemInstruction.html). We send `0x0c` bytes of instruction data:  
  a) 4 bytes [`0x02` padded with zeros] - Indicates we want the `Transfer` [instruction](https://docs.rs/solana-program/1.10.7/solana_program/system_instruction/enum.SystemInstruction.html#variant.Transfer).  
  b) 8 bytes [`input[2]` padded with zeros] - Number of lamports to transfer into destination account  
6. For the `Transfer` call, the first account in the metadata (`accounts[3]`) is the *funding* account, and the second account in the metadata (`accounts[4]`) is the account receiving the funds. **NOTE:** The *funding* account must be owned by the system program, as only the owner of the funding account can decrement it's lamports (anyone can increment any account, but overall things have to balance at the end of the transaction).
7. Finally, there is a bit of book keeping. We modify `accounts[2]`'s data (which only the owner can do, but that was already verified). We index into the data by `0x10` * `input[1]`, and increment the second 8 bytes at that offset by the transferred amount.

This is interesting. There are clearly some flaws here. For one, there are no restrictions on where the transfers go. So, for instance, you could funnel any transaction through this function (including one from and to yourself) and it would increment the requested entry in the ledger account (my name for `accounts[2]`). Also, there is no ownership associated with any of the entries in the ledger account. Anyone can specify any entry. It also turns out there's also **another bug** here which we'll get to later. *Can you spot it*?

### handle_withdraw

<div class="panel-group">
  <div class="panel panel-info">
    <div class="panel-heading">
      <h3 class="panel-title">
        <a data-toggle="collapse" href="#collapseCode4" aria-expanded="false" aria-controls="collapseCode4">
        Ghidra output for handle_withdraw <small>(click to expand)</small>
        <i class="glyphicon glyphicon-menu-left pull-right"></i>
        <i class="glyphicon glyphicon-menu-down pull-right"></i>
        </a>
      </h3>
    </div>
    <div id="collapseCode4" class="panel-collapse collapse">
<div class="panel-body" markdown="1">

```c
undefined8 handle_withdraw(SolParameters *param_1)
{
  uint8_t uVar1;
  uint8_t uVar2;
  bool bVar3;
  long lVar4;
  long lVar5;
  ulong uVar6;
  uint64_t *this_ledger;
  SolInstruction instruction;
  uint8_t inst_buf [12];
  SolAccountMeta meta [2];
  SolSignerSeeds seeds;
  SolSignerSeed seed;
  char seed_buf [6];
  SolAccountInfo (*accounts) [5];
  uint32_t *in_data;
  SolPubkey *account_2_owner;
  uint amount_withdrawn;
  SolPubkey *local_pubkey;
  uint64_t previously_withdrawn;
  
  sol_log(s_handle_withdraw_00013dd0,0xf);
  if (param_1->ka_num != 5) {
    sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x83);
  }
  accounts = (SolAccountInfo (*) [5])param_1->ka;
  local_pubkey = (*accounts)[1].key;
  instruction.data = (uint8_t *)0x0;
  instruction.account_len = 0;
  instruction.accounts = (SolAccountMeta *)0x0;
  instruction.program_id = (SolPubkey *)0x0;
  if (local_pubkey->x[0] == '\0') {
    uVar6 = 1;
    do {
      uVar1 = local_pubkey->x[uVar6];
      uVar2 = *(uint8_t *)((long)&instruction.program_id + uVar6);
      if (uVar1 != uVar2) break;
      bVar3 = uVar6 < 0x1f;
      uVar6 = uVar6 + 1;
    } while (bVar3);
    if (uVar1 != uVar2) goto lbl500ed;
  }
  else {
lbl500ed:
    sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x89);
  }
  local_pubkey = param_1->program_id;
  account_2_owner = (*accounts)[2].owner;
  if (account_2_owner->x[0] == local_pubkey->x[0]) {
    uVar6 = 0;
    if (account_2_owner->x[1] == local_pubkey->x[1]) {
      uVar6 = 0;
      do {
        if (uVar6 == 0x1e) goto lbl501cb;
        lVar4 = uVar6 + 2;
        lVar5 = uVar6 + 2;
        uVar6 = uVar6 + 1;
      } while (account_2_owner->x[lVar4] == local_pubkey->x[lVar5]);
    }
    if (0x1e < uVar6) goto lbl501cb;
  }
  sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x44);
lbl501cb:
  if ((*accounts)[3].is_signer == false) {
    sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x8e);
  }
  if (param_1->data_len - 4 < 0xc) {
    sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x8f);
  }
  in_data = (uint32_t *)param_1->data;
  if (0x280 < in_data[1]) {
    sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x92);
  }
  if (in_data[2] == 0) {
    sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0x93);
  }
  seed_buf[4] = 't';
  seed_buf._0_4_ = 0x6c756176;
  seed_buf[5] = *(char *)(in_data + 3);
  seed.len = 6;
  seed.addr = (uint8_t *)seed_buf;
  seeds.addr = &seed;
  seeds.len = 1;
  meta[0].pubkey = (*accounts)[4].key;
  meta[0]._8_2_ = 0x101;
  meta[1].pubkey = (*accounts)[3].key;
  meta[1]._8_2_ = 1;
  inst_buf._2_2_ = 0;
  inst_buf._0_2_ = 2;
  inst_buf._4_6_ = (uint6)in_data[2];
  inst_buf._10_2_ = 0;
  instruction.program_id = (*accounts)[1].key;
  instruction.data_len = 0xc;
  instruction.data = inst_buf;
  instruction.account_len = 2;
  instruction.accounts = meta;
  sol_invoke_signed_c(&instruction,param_1->ka,(int)param_1->ka_num,&seeds,1);
  this_ledger = (uint64_t *)((*accounts)[2].data + (ulong)in_data[1] * 0x10);
  amount_withdrawn = in_data[2];
  previously_withdrawn = *this_ledger;
  *this_ledger = previously_withdrawn + amount_withdrawn;
                    /* total deposits should exceend total withdrawls */
  if (this_ledger[1] < previously_withdrawn + amount_withdrawn) {
    sol_panic(s_./src/solfire/solfire.c_00013d76,0x18,0xaf);
  }
  return 0;
}
```

</div>
    </div>
  </div>
</div>

This function has a lot of similarities with `handle_deposit`.

1. Make sure we've passed in exactly 5 accounts.
2. Make sure `accounts[1]` key is all zeros. (In b58 encoding, all zeros is represented as `11111111111111111111111111111111` - which is the [system program](https://docs.solana.com/developing/runtime-facilities/programs#system-program)).
3. Make sure `accounts[2]` has an owner and the public key matches `program_id` (ie: that account should be owned by the current program). **NOTE**: I will refer to `accounts[2]` as the "ledger" account.
4. `accounts[3]` **MUST** have signed the request
5. There should be at least 16 bytes of data
6. Treat the input data as an array of ints. `in_data[1]` **MUST** be less than or equal to `0x280`. `in_data[2]` cannot be zero. (Recall: `in_data[0]` was `2` if handle_withdraw was called from `solfire`). Only the first byte of `in_data[3]` is used, and it is used to construct the seed needed to sign the transaction.
7. Construct some account metadata, where the first account is `accounts[4]` and the second account is `accounts[3]`. `accounts[4]` is a signer and both accounts must have `WRITE` permissions.
8. Do a cross-program invocation to the [SYSTEM program](https://docs.rs/solana-program/1.10.7/solana_program/system_instruction/enum.SystemInstruction.html). We send `0x0c` bytes of instruction data:  
  a) 4 bytes [`0x02` padded with zeros] - Indicates we want the `Transfer` [instruction](https://docs.rs/solana-program/1.10.7/solana_program/system_instruction/enum.SystemInstruction.html#variant.Transfer).  
  b) 8 bytes [`in_data[2]` padded with zeros] - number of lamports to transfer into the new account  
6. For the `Transfer` call, the first account in the metadata (`accounts[4]`) is the *funding* account, and the second account in the metadata (`accounts[3]`) is the account receiving the funds. There is extra logic this time around because the program is signing the request on behalf of the vault account (the vault account is a program-derived-address, which means that only this program can sign for it, but it must use the correct seed associated with that particular account). Since we go through the processes of signing on behalf of the vault account, and only the first account needs to be signed, we can surmise that `accounts[4]` *should* be the vault account.
7. Finally, there is a bit of book keeping. We modify `accounts[2]`'s data (which only the owner can do, but that was already verified). We index into the data by `0x10` * `in_data[1]`, and increment the *first* 8 bytes at that offset by the transfered amount. We then verify that the new total of the first 8 bytes does not exceed the value of the second 8 bytes. We can take this to mean that the total withdrawn in this entry should not exceed the total deposited (recall that the second 8 bytes was incremented in `handle_deposit`).

## Some common threads

Notice in the above code there is a common calling pattern for all of the solfire functions. The accounts are expected in the following order:

0. The `C1ock` address
1. The System address
2. The ledger address
3. The user address
4. The vault address

So, for instance, `accounts[2]` is always the ledger account (which is expected to be generated by a call to `handle_create`). `handle_deposit` transfers *from* `accounts[3]` (user - must be signed) into `accounts[4]` (could be any). `handle_withdraw` transfers into `accounts[3]` (could be any, but we probably want it to be user) *from* `accounts[4]` (vault - must be signed). The system address is used to initiate cross-program invocations.

In [Part II of this series]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits/solfire/part-2.md %}), we will look at setting up a test environment with some debug logging and deploying our very own smart-contract.

I. [Part I - Reversing the Binary]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits/solfire.md %}) (you are here)  
II. [Part II - Environment Setup]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits/solfire/part-2.md %})  
III. Part III - Exploitation (*coming soon*)

Or, if you want to read about other challenges, head back to the [picoCTF 2022 Greatest Hits Guide]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits.md %}).

