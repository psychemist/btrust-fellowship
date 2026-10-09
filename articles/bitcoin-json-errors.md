# Why Does This Bitcoin Core RPC Argument Need Quotes?

I am currently midway through my Btrust Open Source Fellowship. This month, we have been interacting with Bitcoin Core using RPC commands, running functional and unit tests, reading and reviewing pull requests, and learning how best to kickoff contributing in the Bitcoin ecosystem.

By the by, over these past few weeks, I spent a reasonable amount of time tracing Bitcoin Core CLI commands from the RPC into the codebase, seeing where exactly the commands are implemented. I must have run up to 50 commands overall, ranging from commands that printed info about the blockchain, node, and network to commands used for creating and manipulating wallets, addresses, and transactions.

Sometimes, I noticed that I would get errors after executing some of these commands. They were usually of the form:

```
error: Error parsing JSON: <value>
```

From the error messages, I could infer that the issue was from sending strings that were not valid JSON input. I was curious why I would get these issues with some commands and not get them with others. I went down a rabbit hole to get to the bottom of the matter.

I will attempt to recreate that journey in this article.

Start your bitcoin daemon with this command:

```sh
bitcoind -regtest daemon
```

You can start the node in any network of your choice.
Run these four commands to return the same block hash on a running regtest node:

```sh
bitcoin-cli -regtest getblockhash 0
bitcoin-cli -regtest getblockhash '0'
bitcoin-cli -regtest getblockhash "0"
bitcoin-cli -regtest getblockhash '"0"'
```

Notice that the last one fails with an error message.

The difference between them is in how the shell processes each of them. In the second and third commands, the shell removes the single and double quotes and passes the text `0` to `bitcoin-cli`. In the fourth, it removes the single quotes but preserves the double quotes inside them. So, the client receives the text `"0"`.

`bitcoin-cli` parses this argument as JSON. If it receives a `0`, it turns it a number. It turns `"0"` to a string based on JSON rules. This specific RPC command takes a numeric block height for, so the server rejects the string input. You can find the JSON parser in the [client conversion code](https://github.com/bitcoin/bitcoin/blob/acdb32693417c1b6596edfbdd19ab793c010d34f/src/rpc/client.cpp) and the RPC command logic in the [getblockhash implementation](https://github.com/bitcoin/bitcoin/blob/acdb32693417c1b6596edfbdd19ab793c010d34f/src/rpc/blockchain.cpp).

### Prerequisite

For the scripts below, I used or Zsh on macOS but you can use Bash on Linux and any other combinations of the four. 

We will be running the rest of these commands in the same shell so the variables and function remain available. Copy and paste this entire script into your terminal.

```sh
rpc_dir=$(mktemp -d)

touch "$rpc_dir/bitcoin.conf"

bitcoind -regtest -datadir="$rpc_dir" \
  -nosettings -listen=0 -networkactive=0 \
  -rpcport=29457 -daemonwait

rpc() {
  bitcoin-cli -regtest -datadir="$rpc_dir" \
    -rpcport=29457 "$@"
}
```

The first line creates a variable that stores the temporary RPC directory. Then, we create a configuration file for the Bitcoin node and save it in that directory. We start a new Bitcoin node using certain configurations and create an `rpc` wrapper scoped to the same directory and port on every call. `"$@"` preserves the arguments you pass to it from the shell. If port 29457 is occupied, choose another port in both places.

Now repeat the opening commands:

```sh
rpc getblockhash 0
rpc getblockhash '0'
rpc getblockhash "0"
rpc getblockhash '"0"'
```

The first three commands return regtest's genesis block hash:

```text
0f9188f13cb7b2c71f2a335e3a4fc328bf5beb436012afca590b1a11466e2206
```

The fourth produces an error with the message:

```text
"Position 1 (height)": "JSON value of type string is not of expected type number"
```

### An Argument's Journey Through Three Layers

These commands follows this path:
- From the terminal commands, they are processed by the shell. The shell decides which characters reach our program and where each argument begins and ends. 
- The argument texts are received by bitcoin-cli and it decides whether to keep an argument as a string or parse it as JSON. This decision depends on the RPC method and parameter
- A JSON request is created containing strictly typed values and the node checks it before executing the RPC. Server validation is run before RPC logic in the Bitcoin Core implementation follows
- The result of the code execution is sent back as a JSO response, or an error if it failed

### Making the Conversion Visible

Bitcoin Core has two testing RPCs, `echo` and `echojson`, which return their arguments. Their server behavior is the same while the client enables JSON conversion for `echojson`. They are useful for this experiment, although they are light testing commands rather than an application API. Here is where you can read their [implementation and help text](https://github.com/bitcoin/bitcoin/blob/acdb32693417c1b6596edfbdd19ab793c010d34f/src/rpc/node.cpp).

Run:

```sh
rpc echo 0 false '["a","b"]'
```

The result is an array containing three strings. I will compact each output into one line to make their types easier to compare 

```json
["0", "false", "[\"a\",\"b\"]"]
```

Now change only the method:

```sh
rpc echojson 0 false '["a","b"]'
```

The result contains a number, a boolean, and an array:

```json
[0, false, ["a", "b"]]
```

The shell supplied the same argument text in both cases. The conversion rules behind each method produced different JSON values.

You can also distinguish a boolean from a string explicitly:

```sh
rpc echojson 'false' '"false"'
```

```json
[false, "false"]
```

The outer single quotes are stripped by the shell. The inner double quotes in the second argument are valid JSON syntax and survive the journey to the client.

### Why Do JSON Objects Even Need Shell Quotes?

Let's take a look at this deliberately broken command:

```sh
rpc echojson "{"data":"6869"}"
```

In Bash and Zsh, the double quotes in that expression open and close shell quoting. They are removed during processing. The client receives:

```text
{data:6869}
```

That is invalid JSON because the object key lacks double quotes. The client reports:

```text
error: Error parsing JSON: {data:6869}
```

If you protect the JSON with single quotes like so:

```sh
rpc echojson '{"data":"6869"}'
```

Now the double quotes reach the JSON parser, and the result is:

```json
[{"data": "6869"}]
```

This same technique keeps an array containing spaces together as one shell argument:

```sh
rpc echojson '["hello world",false,0]'
```

```json
[["hello world", false, 0]]
```

Here, the outer array is the RPC's returned argument list. Its single element is the array you supplied.

This is very important to note because I encountered majority of the JSON errors while interacting with arrays as input.

Single quotes are convenient because they preserve the double quotes that JSON requires. When shell variable expansion is needed, double quotes can work, but embedded JSON quotes must be escaped. For larger structures, it is wiser to allow a JSON library construct the value instead of assembling it by hand.

### Apply it to a Transaction RPC

This command constructs a transaction containing a data output:

```sh
rpc createrawtransaction '[]' '{"data":"6869"}' 0 false
```

Each argument in this command has a different role:

| Parameter     | Text received by the client | JSON value                      |
| ------------- | --------------------------- | ------------------------------- |
| `inputs`      | `[]`                        | Empty array                     |
| `outputs`     | `{"data":"6869"}`           | Object describing a data output |
| `locktime`    | `0`                         | Number                          |
| `replaceable` | `false`                     | Boolean                         |
