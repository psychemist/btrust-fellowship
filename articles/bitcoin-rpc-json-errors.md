# Why Does This Bitcoin Core RPC Argument Need Quotes?

I am currently midway through my Btrust Open Source Fellowship. This month, we have been interacting with Bitcoin Core using RPC commands, running functional and unit tests, reading and reviewing pull requests, and learning how best to kickoff contributing in the Bitcoin ecosystem.

By the by, over these past few weeks, I spent a reasonable amount of time tracing Bitcoin Core CLI commands from the RPC into the codebase, seeing where exactly the commands are implemented. I must have more than 50 commands overall, ranging from commands that printed info about the blockchain, node, and network to commands used for creating and manipulating wallets, addresses, and transactions.

Sometimes, I noticed that I would get errors after executing some of these commands. They were usually of the form:

```
error: Error parsing JSON: <value>
```

From the error messages, I could infer that the issue was from sending strings that were not valid JSON input. I was curious why I would get these issues with some commands and not get them with others. I went down a rabbit hole to get to the bottom of the matter.

I will attempt to recreate that journey in this article.

Start your bitcoin daemon with this command:

```sh
bitcoind -regtest -daemon
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

`bitcoin-cli` parses this argument as JSON. If it receives a `0`, it turns it a number. It turns `"0"` to a string based on JSON rules. This specific RPC command takes a numeric block height for, so the server rejects the string input. You can find the RPC command logic in the [getblockhash implementation](https://github.com/bitcoin/bitcoin/blob/acdb32693417c1b6596edfbdd19ab793c010d34f/src/rpc/blockchain.cpp).

### Prerequisite Scripting

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

The first line creates a variable that stores the temporary RPC directory. Then, we create a configuration file for the Bitcoin node and save it in that directory. We start a new Bitcoin node using certain configurations and create a bitcoin-cli `rpc` wrapper function scoped to the same directory and port on every call. `"$@"` preserves the arguments you pass to the function from the shell. If port 29457 is occupied o your machine, choose another port in both places.

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

The commands follows this execution path:
- From the terminal commands, they are processed by the shell. The shell decides which characters reach our small program and where each argument begins and ends. 
- From our wrapper, the argument texts are received by bitcoin-cli and it decides whether to keep an argument as a string or parse it as JSON. This decision depends on the RPC method and parameter.
- A JSON request is created containing strictly typed values and the node checks it before executing the RPC. Server validation is run before the RPC logic in the Bitcoin Core implementation follows
- The result of the code execution is sent back as a JSON response, or an error if it failed

### Making the Conversion Visible

Bitcoin Core has two testing RPC methods, `echo` and `echojson`, which return their arguments. Their server behavior is the same while the client enables JSON conversion for `echojson`. They are useful for this experiment, although they are light testing commands rather than an application API. Here is where you can read their [implementation and help text](https://github.com/bitcoin/bitcoin/blob/acdb32693417c1b6596edfbdd19ab793c010d34f/src/rpc/node.cpp).

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

The shell supplied the same argument text in both cases. The conversion rules behind each method produced different JSON values. Note that while `echo` returns an array with values escaped using the `\` character, `echojson` prints a valid, formatted JSON array. 

These commands can also explicitly distinguish booleans from strings:

```sh
rpc echojson 'false' '"false"'
```

```json
[false, "false"]
```

The outer single quotes are stripped by the shell. The inner double quotes in the second argument are valid JSON syntax and survive the journey to the client. Run it with the other method and spot the differences between the two array elements.

### Why Do JSON Objects Even Need Shell Quotes?

Let's take a look at this deliberately broken command:

```sh
rpc echojson "{"data":"6869"}"
```

In Bash and Zsh, the double quotes in that expression open and close shell quoting. They are removed during processing. The client receives: `{data:6869}`.

That is invalid JSON because the object key lacks double quotes. The client returns:

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
rpc echojson '["hello world", false, 0]'
```

```json
[["hello world", false, 0]]
```

Here, the outer array is the RPC's returned argument list. Its single element is the array we supplied as argument to the command.

Single quotes are convenient because they preserve the double quotes that JSON requires. When shell variable expansion is needed, double quotes can work, but JSON quotes that are embedded in the argument list must be escaped. For larger input structures, it is easier to allow a JSON library construct the value instead of assembling them by hand.

This is very important to note because I encountered majority of the JSON errors while interacting with arrays as input.

### Apply it to a Transaction RPC

Let's apply this to a real RPC method: `createrawtransaction`. 

This command accepts two required JSON array arguments, _inputs and outputs_, and two optional arguments, _locktime and replaceable_, a numeric and a boolean type respectively. Itcreates a transaction, spending the given inputs and creating new outputs, which can be addresses or data, and returns a hex-encoded raw transaction.

The examples below pays 0.01 BTC to a regtest address and returns 49.9899 BTC to a change address, leaving a fee of 0.0001 BTC from a 50 BTC input. The values shown here are from a regtest wallet. The input was a mature, unspent coinbase output while the payment and change addresses are actual regtest addresses.

First, run the command with the wrong JSON type for `replaceable`:

```sh
rpc createrawtransaction \
  '[{"txid":"e4813b21094e242874de0bc4f7b86ad6e26f1d9504c3b1375356a2f0f4ad7518","vout":0}]' \
  '{"bcrt1qn0u0s7w5lt49zxflp6095y0c74mh2p7fm2vjs6":0.01,"bcrt1qxrus33z8c2ku78jn7nuqj92k9rj6fl46p5a2pu":49.9899}' \
  0 '"false"'
```

It fails with:

```text
error code: -3
error message:
Wrong type passed:
{
    "Position 4 (replaceable)": "JSON value of type string is not of expected type bool"
}
```

The outer single quotes preserve the double quotes around `false`. The client parses the remaining text `"false"` as a JSON string. The server expects a boolean type for `replaceable`, so it rejects the request. The other arguments are correctly formed.

Now change only the final argument:

```sh
rpc createrawtransaction \
  '[{"txid":"e4813b21094e242874de0bc4f7b86ad6e26f1d9504c3b1375356a2f0f4ad7518","vout":0}]' \
  '{"bcrt1qn0u0s7w5lt49zxflp6095y0c74mh2p7fm2vjs6":0.01,"bcrt1qxrus33z8c2ku78jn7nuqj92k9rj6fl46p5a2pu":49.9899}' \
  0 false
```

This succeeds and returns the unsigned transaction hex:

```text
02000000011875adf4f0a2565337b1c304951d6fe2d66ab8f7c40bde7428244e09213b81e40000000000ffffffff0240420f00000000001600149bf8f879d4faea51193f0e9e5a11f8f5777507c9b088f6290100000016001430f908c447c2adcf1e53f4f809155628e5a4feba00000000
```

Without the preserved JSON double quotes, the client correctly parses `false` as a boolean. The RPC can then construct the transaction from the given input and output.

Next, we repeat this test for the inputs parameter. The `inputs` argument is a JSON array. Each object inside the array is linked to a transaction output using its prevout i.e `txid` and `vout`.

Consider this command, where the input array has no surrounding shell quotes:

```shell
rpc createrawtransaction \
  [{"txid":"e4813b21094e242874de0bc4f7b86ad6e26f1d9504c3b1375356a2f0f4ad7518","vout":0}] \
  '{"bcrt1qn0u0s7w5lt49zxflp6095y0c74mh2p7fm2vjs6":0.01,"bcrt1qxrus33z8c2ku78jn7nuqj92k9rj6fl46p5a2pu":49.9899}' \
  0 false
```

This fails before the node receives the RPC request. With default shell settings:
- **Zsh** treats the brackets as a filename matching pattern and reports `no matches found`.
- **Bash** expands the braces and removes the double quotes. The client then receives malformed arguments and reports `Error parsing JSON`.

Now add the single quotes back around the input array, leaving every other argument unchanged:

```sh
rpc createrawtransaction \
  '[{"txid":"e4813b21094e242874de0bc4f7b86ad6e26f1d9504c3b1375356a2f0f4ad7518","vout":0}]' \
  '{"bcrt1qn0u0s7w5lt49zxflp6095y0c74mh2p7fm2vjs6":0.01,"bcrt1qxrus33z8c2ku78jn7nuqj92k9rj6fl46p5a2pu":49.9899}' \
  0 false
```

This succeeds and returns unsigned transaction hex. The outer single quotes keep the array together as one shell argument and preserve its internal JSON double quotes. The shell removes those outer quotes then `bitcoin-cli` receives a valid JSON input and converts it into an array for the request.

If you repeat this with the output array, it will error with this message: `error: Error parsing JSON` because the shell removes the JSON double quotes and expands the comma-separated text into separate arguments, leading to incorrectly parsed JSON. 

The values used in these tests are fixed and obtained from my wallet. To test these on transactions using your own wallet, obtain an unspent input transaction with `listunspent`, a payment address with `getnewaddress`, and a change address with `getrawchangeaddress`. Substitute that input's actual `txid` and `vout`, and calculate change from its amount, the payment, and the intended fee. 

The command's construction behavior is described in the [RPC reference](https://developer.bitcoin.org/reference/rpc/createrawtransaction.html/).

### Named Arguments

Names help make bitcoin-cli commands readable and allow you to supply optional parameters without filling every earlier position. 

```sh
rpc -named echojson arg0='{"data":"6869"}' arg1=false
```

Output:

```json
[{"data": "6869"}, false]
```

They do not remove the need to preserve JSON syntax. Without the single quotes in argo, we get an error:

```sh
rpc -named echojson arg0={"data":"6869"} arg1=false
```

Output:

```
error: Error parsing JSON: {data:6869}
```

For a better example, with `createrawtransaction`,  instead of the fixed positions, you can send this command with the arguments out of order:

```sh
rpc -named createrawtransaction replaceable=false locktime=0 \

'[{"txid":"e4813b21094e242874de0bc4f7b86ad6e26f1d9504c3b1375356a2f0f4ad7518","vout":0}]' \

'{"bcrt1qn0u0s7w5lt49zxflp6095y0c74mh2p7fm2vjs6":0.01,"bcrt1qxrus33z8c2ku78jn7nuqj92k9rj6fl46p5a2pu":49.9899}'
```

Output:

```
02000000011875adf4f0a2565337b1c304951d6fe2d66ab8f7c40bde7428244e09213b81e40000000000ffffffff0240420f00000000001600149bf8f879d4faea51193f0e9e5a11f8f5777507c9b088f6290100000016001430f908c447c2adcf1e53f4f809155628e5a4feba00000000
```

Note that I mixed both named and unnamed arguments and still got a valid txid.

### String Conversion

For an ordinary string parameter, you can also supply text directly without quotes. To demo this, we will create a new wallet using a name without quotes then request an address with a quoted-enclosed label that contains a space:

```sh
rpc createwallet quotes-demo

rpc -rpcwallet=quotes-demo -named getnewaddress \
  label='wallet demo' address_type=bech32
```

Output:

```json
{ "name": "quotes-demo" } 

bcrt1qkfmwy6a2mmaqza37amau3c50yfgcgrd4yt3hpd
```

The wallet name is converted automatically to a string parameter. The label reaches the client as one argument and becomes the string `wallet demo`. You do not need to add JSON double quotes around them both. 

This is a handy table for tracking how each argument type is handled by the client based on the  [conversion rules](https://github.com/bitcoin/bitcoin/blob/acdb32693417c1b6596edfbdd19ab793c010d34f/src/rpc/client.cpp).

| Format           | Client behavior                                                           |
| ---------------- | ------------------------------------------------------------------------- |
| `JSON`           | Parse the argument as JSON; report an error if parsing fails              |
| `STRING`         | Preserve the argument as a string.                                        |
| `JSON_OR_STRING` | Try JSON parsing; preserve the original text as a string if parsing fails |

An argument without a matching conversion entry is also preserved as a string. Explicit string entries additionally help differentiate between positional arguments and named arguments  containing `=` .

### Bonus Exercise

As a final exercise, predict the result before running:

```sh
rpc echojson 0 '0' '"0"' false '"false"' '[]'
```

Then replace `echojson` with `echo` and explain why the types change. Use the conversion table to check your explanation. 

Before changing quotes at random, ask what text reached the client and what value it produced using this table:

| Symptom                                               | What to Investigate                                    |
| ----------------------------------------------------- | ------------------------------------------------------ |
| Shell error or unexpected argument count              | Quoting, expansion, and argument boundaries            |
| Client reports `Error parsing JSON`                   | Whether the preserved argument is valid JSON           |
| Server reports a type mismatch                        | Whether the conversion produced the expected JSON type |
| Server rejects a value such as an out-of-range height | The RPC's rules for that value                         |

### Final Words

Your next RPC interactions can start with the same sequence we took: identify the shell arguments, inspect its client conversion code, and read the server's RPC parameter definition. Hopefully, this helps you understand errors and make less of them as you continue your Bitcoin journey.

If you started a regtest node specifically for this exercise, stop it when you finish:

```sh
rpc stop
```

---