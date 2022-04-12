---
title:  picoCTF 2022 - Binary Exploitation - Solfire (500 points) - Part II
description: Setting up a Solana development environment, adding logging, and deploying our first smart contract.
hide_index: true
image:
  path: /img/reference/pico-ctf-2022-00000.png
  width: 1024
  height: 512
  thumb: /img/reference/pico-ctf-2022-00000.thumb.png
  alt: Linux Terminal
---

# picoCTF 2022 - Solfire (Part II)

> **Note**: This article is part of our [picoCTF 2022 Greatest Hits Guide]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits.md %}).

This is the second part of a *three part series*. In Part II, we will cover setting up your test environment and deploying an eBPF binary.

I. [Part I - Reversing the Binary]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits/solfire.md %})  
II. [Part II - Environment Setup]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits/solfire/part-2.md %}) (you are here)  
III. Part III - Exploitation (*coming soon*)

## The Problem

We are given a Dockerfile. It is a little slow to build, but eventually it works. However, when running the instance we get almost no feedback about what's going on. We need to increase the verbosity of what this program is doing so we can figure out what is happening (and what is going *wrong*). Ideally, we'd like a way to make some changes *without* it taking forever to build again.

Let's modify the Dockerfile like so:

```diff
--- Dockerfile.orig     2022-03-12 14:35:13.000000000 -0700
+++ Dockerfile  2022-04-12 06:00:00.000000000 -0600
@@ -13,6 +13,12 @@

 COPY solfire.so ./
 
+COPY newsrc/main.rs ./src/main.rs
+RUN touch src/main.rs && cargo build --release
+ENV RUST_BACKTRACE=full
+
 CMD [ "./target/release/solfire" ]


+# docker build -t 'picoctf2022-solfire' .
+# docker run --rm -p 8080:8080 picoctf2022-solfire
```

Let's also create a copy of main.rs inside of a directory named `newsrc` and apply the following edits:
```diff
--- src/main.rs 2022-03-12 14:55:51.000000000 -0700
+++ newsrc/main.rs      2022-04-12 06:00:00.000000000 -0600
@@ -18,12 +18,17 @@
 use poc_framework::{
     solana_sdk::{self, signature::Keypair, signer::Signer},
     Environment, LocalEnvironment,
+    setup_logging, LogLevel,
+    PrintableTransaction,
 };
 use std::env;

 fn main() -> Result<(), Box<dyn Error>> {
     let listener = TcpListener::bind("0.0.0.0:8080")?;
     let pool = ThreadPool::new(4);
+
+    setup_logging(LogLevel::DEBUG);
+
     for stream in listener.incoming() {
         let stream = stream.unwrap();

@@ -79,7 +84,7 @@
         )
         ],
         &[&env.payer()],
-    );
+    ).print();

     // .message.serialize().len()

@@ -129,7 +134,7 @@
         env.get_recent_blockhash(),
     );
 
-    env.execute_transaction(tx);
+    env.execute_transaction(tx).print();
     let user_bal = env.get_account(user.pubkey()).unwrap().lamports;
     writeln!(socket, "user bal: {:?}", user_bal)?;
     writeln!(socket, "vault bal: {:?}", env.get_account(vault).unwrap().lamports)?;
```

What the heck do these edits do? Well, the first one will add some extra steps to the Dockerfile right after everything has been built once. The great thing about doing it this way is that we will leverage docker's build cache, so that we replace the `main.rs` file only *after* it's already built everything, meaning docker can re-use it's existing cache of everything up until that step. We then force cargo to rebuild again, which is pretty fast since the only thing that has changed is one file.

What edits did we make to `main.rs`? Nothing major, we just followed the hints suggested on the [poc_framework](https://github.com/neodyme-labs/solana-poc-framework) github - turn on logging with a loglevel of `DEBUG`, and modify the transactions to call `.print()` after executing. We also set `RUST_BACKTRACE=full` inside the docker container, which may produce more meaningful callstacks in some cases.

## Helloworld

What next? Well, somehow we need to deploy our own smart-contract. To do that, we need to figure out how to use the Solana toolchain.

Since Rust isn't something we're very familiar with, we're going to use the C sdk. Unfortunately, I found the [official documentation](https://docs.solana.com/developing/on-chain-programs/developing-c) to be somewhat lacking. There are an additional 2 critical pieces of information needed:

1. Your project structure has to be exactly `/src/<name>/<name>.c`. This is actually how their github is setup, but because of the way github merges `src/<name>` as one, it's not necessarily obvious if you were just trying to re-create it by hand.
2. The instructions do not cover an extra mandatory step: you must cd into `~/.local/share/solana/install/active_release/bin/sdk/bpf/` and run `env.sh` in order to install all the required development tools.

Here's a Dockerfile to get you a working Solana C development environment:

```Dockerfile
FROM ubuntu

RUN apt-get update \
    && apt-get install -y ca-certificates curl make \
      --no-install-recommends \
    && rm -rf /var/lib/apt/lists/* \
    && (curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y) \
    && . $HOME/.cargo/env \
    && sh -c "$(curl -sSfL https://release.solana.com/v1.10.8/install)" \
    && export PATH="/root/.local/share/solana/install/active_release/bin:$PATH" \
    && (cd ~/.local/share/solana/install/active_release/bin/sdk/bpf/ && sh ./env.sh)

WORKDIR /work
ENV PATH="/root/.local/share/solana/install/active_release/bin:$PATH" 

# docker build -t 'solana-dev' .
# docker run --rm -it -v $PWD:/work solana-dev make
```

Here's a Makefile to put in that directory:

```Makefile
OUT_DIR := ./dist
include ~/.local/share/solana/install/active_release/bin/sdk/bpf/c/bpf.mk
```

And here's a helloworld program to put inside `src/helloworld/helloworld.c` (It **must** be exactly this path):

```c
/**
 * @brief C-based Helloworld BPF program
 */
#include <solana_sdk.h>

uint64_t helloworld(SolParameters *params)
{
  sol_log("Hello!");

  return SUCCESS;
}

extern uint64_t entrypoint(const uint8_t *input) {
  sol_log("Helloworld C program entrypoint");

  SolAccountInfo accounts[1];
  SolParameters params = (SolParameters){.ka = accounts};

  if (!sol_deserialize(input, &params, SOL_ARRAY_SIZE(accounts))) {
    return ERROR_INVALID_ARGUMENT;
  }

  return helloworld(&params);
}
```

If you want, you can download all of those together in this file: [helloworld.tgz]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits/solfire/helloworld.tgz %}).

To use simply build the image once:

```
$ docker build -t 'solana-dev' .
[+] Building 0.2s (7/7) FINISHED
 => [internal] load build definition from Dockerfile
 => => transferring dockerfile: 38B
 => [internal] load .dockerignore
 => => transferring context: 2B
 => [internal] load metadata for docker.io/library/ubuntu:latest
 => [1/3] FROM docker.io/library/ubuntu
 => CACHED [2/3] RUN apt-get update     && apt-get install -y ca-certificates curl make       --no-install-recommends     && rm -rf /var/lib/apt/lists/*     && (curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s --
 => CACHED [3/3] WORKDIR /work
 => exporting to image
 => => exporting layers
 => => writing image sha256:2ebd235cfccfc73b0e2ef413d2d29d5b18f2b510402742a7fdb192e59db2455e
 => => naming to docker.io/library/solana-dev

Use 'docker scan' to run Snyk tests against images to find vulnerabilities and learn how to fix them
```
{:.contains-term}

and then use the image and `make` the projects in the current directory every time you want to recompile:

```
$ docker run --rm -it -v $PWD:/work solana-dev make
[cc] ./dist/helloworld/helloworld.o (./src/helloworld/helloworld.c)
[lld] ./dist/helloworld.so (./dist/helloworld/helloworld.o)
Wrote new keypair to ./dist/helloworld-keypair.json
To deploy this program:
$ solana program deploy /work/dist/helloworld.so
```
{:.contains-term}

which will create a new file `dist/helloworld.so` which is our eBPF binary to deploy.

## Deploying our binary

Last but not least, we need to actually figure out how to deploy this binary. As per usual, we'll whip something up in pwntools:

```python
#!/usr/bin/env python3

from pwn import *

p = remote('localhost','8080')

with open("./dist/helloworld.so","rb") as f:
    f.seek(0, os.SEEK_END)
    flen = f.tell()
    p.sendline(str(flen))
    f.seek(0, os.SEEK_SET)
    p.send(f.read())

p.readuntil("program pubkey: ")
program_pubkey = p.readline(keepends=False).decode()

p.readuntil("solve pubkey: ")
solve_pubkey = p.readline(keepends=False).decode()

p.readuntil("user pubkey: ")
user_pubkey = p.readline(keepends=False).decode()

# print what we know
print(f"program: {program_pubkey}\nsolve: {solve_pubkey}\nuser: {user_pubkey}")

#metadata
p.sendline("0")

#instructions
buf=b""
p.sendline(str(len(buf)))
p.send(buf)
p.stream()
```

To test it, simply launch and instance of the docker image:

```
$ docker run --rm -p 8080:8080 picoctf2022-solfire
```
{:.contains-term}

and then in another terminal, run our new python script:

```
$ python3 connect.py
[+] Opening connection to localhost on port 8080: Done
program: Ew7GBvH4DQyPF7SMdV398ymLoDpYLgiHg4TNBWwee6Da
solve: 4uJfeKUTXmuiy3oLkDoB98TKGiCtE7qfNXPcYYbXwzQi
user: 64tZQcNh2PrWRteUVwPt3x7oDUM7vEkaLmkdcbKPGeSn
user bal: 10
vault bal: 1000000
```
{:.contains-term}

If you switch back to the docker instance, you should see a really *long* log output, containing (toward the end) something that looks like this:
```
...
EXECUTE  (slot 0)
  Recent Blockhash: 94N8zHwhUWhChUbBQrLShL2xWB22ENuXvVDgmsBn76yd
  Signature 0: jrSVMEACSrQVwMg52iJwv18GP3xEcgCxq6w72St6KnGvkvKM2TZNLbUsp7utK5wFwrWqeVTheU4Dr8WinjYEzEP
  Account 0: srw- 64tZQcNh2PrWRteUVwPt3x7oDUM7vEkaLmkdcbKPGeSn (fee payer)
  Account 1: -r-x 4uJfeKUTXmuiy3oLkDoB98TKGiCtE7qfNXPcYYbXwzQi
  Instruction 0
    Program:   4uJfeKUTXmuiy3oLkDoB98TKGiCtE7qfNXPcYYbXwzQi (1)
    Data: []
  Status: Ok
    Fee: ◎0
    Account 0 balance: ◎0.00000001
    Account 1 balance: ◎0.01636992
  Log Messages:
    Program 4uJfeKUTXmuiy3oLkDoB98TKGiCtE7qfNXPcYYbXwzQi invoke [1]
    Program log: Helloworld C program entrypoint
    Program log: Hello!
    Program 4uJfeKUTXmuiy3oLkDoB98TKGiCtE7qfNXPcYYbXwzQi consumed 48 of 200000 compute units
    Program 4uJfeKUTXmuiy3oLkDoB98TKGiCtE7qfNXPcYYbXwzQi success
...
```
{:.contains-term}

Aha! We've confirmed that our program runs! Plus we now have a bunch of useful log information we can use to debug potential problems we might face along the way. **Problems like how to steal 50000 lamports from the vault!**

In Part III of this series (*coming soon*), we will look at how to exploit `solfire.so` to steal those lamports and grab the flag.

I. [Part I - Reversing the Binary]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits/solfire.md %})  
II. [Part II - Environment Setup]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits/solfire/part-2.md %}) (you are here)  
III. Part III - Exploitation (*coming soon*)

Or, if you want to read about other challenges, head back to the [picoCTF 2022 Greatest Hits Guide]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits.md %}).