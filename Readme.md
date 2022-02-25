# Deploying ERC-20 Token to Solana

1. Installing Solang
The best way to install Solang is using the VS Code extension. It will automatically install the correct solang binary along with the dependencies. Alternatively you can download the binary directly and manually install the dependencies. The VS Code extension will also give you compiling capabilities for Solang, the regular Solidity extension wouldn't be quite accurate due to the differences in the supported features.

You'll also want to initialize the package and install required dependencies:

```bash
$ npm init
$ npm install @solana/solidity @solana/web3.js
```

2. Installing Solana Tool Suite
Next up install the Solana test suite. If you're running on Mac OS, it's as simple as running:

```bash
$ sh -c "$(curl -sSfL https://release.solana.com/v1.8.5/install)"
```

3. Create the ERC-20 contract
Now let's take an ERC20 contract as ERC20.sol in our package root, the code here is almost a 1:1 copy from Openzeppelin.

4. Compile Solidity -> Solana
Next up install the Solana test suite. If you're running on Mac OS, it's as simple as running:

```bash
$ solang ERC20.sol --target solana --output build
```

This will produce

build/ERC20.abi: Just as you know from Ethereum, the ABI for our contract is generated.
build/bundle.so: New here is the compiled Solana Program.

5. Deploy the ERC-20 contract

Now create the following deploy-erc20.js script:

```bash
const { Connection, LAMPORTS_PER_SOL, Keypair } = require('@solana/web3.js')
const { Contract, publicKeyToHex } = require('@solana/solidity')
const { readFileSync } = require('fs')

const ERC20_ABI = JSON.parse(readFileSync('./build/ERC20.abi', 'utf8'))
const BUNDLE_SO = readFileSync('./build/bundle.so')

;(async function () {
    console.log('Connecting to your local Solana node ...')
    const connection = new Connection(
        // works only for localhost at the time of writing
        // see https://github.com/solana-labs/solana-solidity.js/issues/8
        'http://localhost:8899', // "https://api.devnet.solana.com",
        'confirmed'
    )

    const payer = Keypair.generate()
    while (true) {
        console.log('Airdropping (from faucet) SOL to a new wallet ...')
        await connection.requestAirdrop(payer.publicKey, 1 * LAMPORTS_PER_SOL)
        await new Promise((resolve) => setTimeout(resolve, 1000))
        if (await connection.getBalance(payer.publicKey)) break
    }

    const address = publicKeyToHex(payer.publicKey)
    const program = Keypair.generate()
    const storage = Keypair.generate()

    const contract = new Contract(connection, program.publicKey, storage.publicKey, ERC20_ABI, payer)

    console.log('Deploying the Solang-compiled ERC20 program ...')
    await contract.load(program, BUNDLE_SO)

    console.log('Program deployment finished, deploying ERC20 ...')
    await contract.deploy('ERC20', ['MyToken', 'MTO', '1000000000000000000'], program, storage, 4096 * 8)

    console.log('Contract deployment finished, invoking contract functions ...')
    const symbol = await contract.symbol()
    const balance = await contract.balanceOf(address)

    console.log(`ERC20 contract for ${symbol} deployed!`)
    console.log(`Wallet at ${address} has a balance of ${balance}.`)

    contract.addEventListener(function (event) {
        console.log(`${event.name} event emitted!`)
        console.log(
            `${event.args[0]} sent ${event.args[2]} tokens to
       ${event.args[1]}`
        )
    })

    console.log('Sending tokens will emit a "Transfer" event ...')
    const recipient = Keypair.generate()
    await contract.transfer(publicKeyToHex(recipient.publicKey), '1000000000000000000')

    process.exit(0)
})()
```

Here we are making use of

Solana Web3
Solana Solidty.

If you are Dapp developer and want to connect a wallet, have a look at Solana wallet adapter instead.

Now we're ready to run your own local Solana chain:

```bash
$ solana-test-validator --reset --quiet
```

And in a separate tab run our script:

```bash
$ node deploy-erc20.js
```

We just deployed the ERC-20 token to our local Solana chain! 🎉🎉🎉