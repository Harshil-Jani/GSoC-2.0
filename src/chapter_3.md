# Coding Version 0 and Challenges Faced

## Kickstarting coding

The very first step in developing the Matrix bridge for qaul was to attempt and create a login functionality into the matrix from the qaul code. As mentioned earlier, By reffering the `examples/` in `matrix-sdk` which was latest at `0.6.2` by the time I am writing this, I wrote a new file inside qaul-matrix-bridge binary called `relay_bot.rs` along with a `connect()` method which will allow us to connect with matrix.

## Dependencies conflicts

With the new `connect()` method and the `relay_bot.rs` code, compiler complaint heavily with list of errors. On debugging, The errors were about dependency version conflict.

With latest `matrix-sdk 0.6.2`, Version mismatched for `libp2p` which at the time has latest `0.51.x` and qaul was using `0.50.0` and `0.52.0` was about to release officially within next upcoming month. The code was compiling if we downgrade `matrix-sdk` to `0.4.0`. But then we lose some methods which connects qaul directly with some abstraction level to matrix.

I reported this to MathJud. He took some time to look at the dependency tree and figured out the exact conflicts. We were using `libp2p` at `0.50.x` and it had `libp2p-swarm-derive`. Now the `matrix-sdk` and `libp2p-swarm-derive` has two packages in common

- quote
- syn

And the `syn` package had a big version number change 1->2.
But If `libp2p` `0.52.0` gets released, It would have solved the conflict but it would require us to do some code changes to upgrade qaul to onboard `0.52.0` of libp2p. So, For inital rounds of test, We decided to stick with an older version of matrix-sdk with no change inside qaul.

I have although asked the maintainers of libp2p about when they'd plan their next release in one of their open issues : [Link to discussion on `lib p2p 0.52.0` release](https://github.com/libp2p/rust-libp2p/issues/3647#issuecomment-1559944456)

## Sailing the Boat

I have now started coding with `matrix-sdk 0.4.0` and implemented the very first feature which listens on the room for any messages on matrix and return us back with a simple message.

The `relay_bot::connect()` method were rightly used for login into matrix and listening on the matrix client for any activity or interact with it. Since this process required synchronus listening on the matrix service, It actually blocked the `qaul-cli` and has no meaning as an implementation. We needed to run both the process simultaneously and synchronously.

- Keep checking for messages on matrix
- Keep checking for messages in qaul

To make both the calls on both side sync, We thought to run the process of matrix login and configuration inside a different thread. This way, the process kept running in new thread without blocking our qaul-cli. Another option was that we may have polled the matrix part inside the loop which runs in qaul for every 10 miliseconds for checking any command line events or RPC events. But, It comes with a problem of overhead since, `relay_bot::connect()` logs in every time we run in a loop and tweaking it was not likely the best option. Running in separate thread was the best thing which we got to see clearly.

```rust
thread::spawn(|| {
        // connect the matrix bot with the qaul-cli
        match relay_bot::connect() {
            Ok(_) => {
                println!("Matrix-Bridge connecting");
            }
            Err(error) => {
                println!("{}", error);
            }
        }
    });
```
