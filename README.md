# Pixel Climb (web)

The browser version of [Pixel Climb](https://play.google.com/store/apps/details?id=com.harvz.pixelclimb), where players earn and spend **$PCMB** on **NEAR testnet** without a wallet popup. MetaMask can also connect an existing wallet.

Built for [NEAR HACKATHON](https://nearhackathon.com/) (1–14 Nov 2026), track **Games With Real Economies**.

**Play it: https://play.pixelclimb.xyz** (NEAR testnet, no install and no wallet needed to start)

Pixel Climb is a one-tap climbing game. You tap to jump between walls, collect gems on the way up and try to beat your best floor. It is already live on Google Play. This repo puts the same game in the browser so players can try it without installing anything, and connects their wallet on NEAR.

The landing site lives in a separate repo, [hsjtechnologies/pixelclimb.xyz](https://github.com/hsjtechnologies/pixelclimb.xyz). Both repos are the same entry.

## How NEAR fits in

The player connects MetaMask, and the app switches it to **NEAR Protocol Testnet** through NEAR's Ethereum-compatible RPC ([NEP-518](https://github.com/near/NEPs/issues/518)):

| Setting  | Value                              |
| -------- | ---------------------------------- |
| Chain ID | `398` (`0x18e`)                    |
| RPC      | `https://eth-rpc.testnet.near.org` |
| Currency | NEAR (18 decimals)                 |
| Explorer | `https://testnet.nearblocks.io`    |

The player's `0x…` address is their NEAR account, so players use the wallet they already have and need no new NEAR account. The code is in [`src/lib/connect-wallet.svelte`](src/lib/connect-wallet.svelte).

### Check it without a wallet

```sh
pnpm verify
```

This reads NEAR testnet and prints the $PCMB supply, the prize pool, the gem price and the round end. It also checks that the token and economy contracts on testnet are the same code as [`contracts/`](contracts) in this repo. It needs no wallet or keys. `pnpm verify mainnet` checks the mainnet contracts the same way. `pnpm verify` needs the same Node as the rest of the repo (22.22.2 or newer). Without pnpm, `node scripts/verify.mjs` does the same on Node 22 and needs no install.

### Economy

The in-game currency is **$PCMB**, a NEP-141 token on NEAR testnet at [`pcmb.pixelclimb.testnet`](https://testnet.nearblocks.io/token/pcmb.pixelclimb.testnet) (5,000,000 supply, 9 decimals). It also serves as the governance token. MetaMask shows it as a normal token at `0x937ca2395d64855e08c74c30947a545eef6c1bf1` on NEAR Protocol Testnet. The contract is in [`contracts/pcmb-token`](contracts/pcmb-token).

The game economy runs in a second contract at [`economy.pixelclimb.testnet`](https://testnet.nearblocks.io/address/economy.pixelclimb.testnet), with the code in [`contracts/economy`](contracts/economy):

- **Earn.** Each round has a prize pool of $PCMB. When the round ends, the pool is split among players who reached at least floor 10, in proportion to their best floor.
- **Spend.** A revive costs $PCMB. The price rises 5% with each purchase and drops 1% a minute when nobody buys.
- **Recycle.** Every $PCMB spent on revives goes into the next round's pool.

How the pool behaves in practice:

- **Rounds last 5 minutes** (a demo setting; 1,440 minutes is the planned production value). A round pays out when someone triggers it: the page does this a few seconds after the countdown ends, and so does the next saved run.
- **Each new round starts with 1,000 $PCMB** from a reserve, plus the revive payments of the round before. The owner funds the reserve; `pnpm verify` shows how much is left.
- **One qualifying player takes the whole pool.** The split is by best floor among everyone who reached floor 10, so a player who is alone in a round gets all of it.
- **If nobody reaches floor 10,** the pool carries over to the next round and takes nothing more from the reserve.
- **Revives are paid from the game balance,** not from the wallet, so they never open MetaMask. **Withdraw** sends the whole game balance to the wallet; **Deposit** moves $PCMB from the wallet back into the game.

To refill the reserve (owner key, near-cli-rs):

```sh
near contract call-function as-transaction pcmb.pixelclimb.testnet ft_transfer_call json-args '{"receiver_id":"economy.pixelclimb.testnet","amount":"100000000000000","msg":"reserve"}' prepaid-gas '100.0 Tgas' attached-deposit '1 yoctoNEAR' sign-as pixelclimb.testnet network-config testnet sign-with-legacy-keychain send
```

That adds 100,000 $PCMB, enough for 100 rounds that have a winner.

### Play first, connect a wallet to claim

A new visitor earns with no wallet. On a player's first visit the page makes an ed25519 key in the browser (`localStorage`). Its public key, in hex, is a NEAR account (an [implicit account](https://docs.near.org/protocol/account-id#implicit-address)), and runs, payouts and revives go to that browser account.

Connecting MetaMask makes the wallet the player's account. NEAR takes the lowercase `0x…` address as an account id ([NEP-518](https://github.com/near/NEPs/issues/518)). On connect the wallet signs one message that lets this browser's key play for it (`personal_sign`), and everything the browser account earned so far (balance, gems, this round's best floor) moves to the wallet. From then on runs, payouts and revives go to the wallet, **Withdraw** sends the balance to it, and **Deposit** sends the wallet's $PCMB back into the game with one MetaMask confirmation (an `ft_transfer_call` to the economy, sent through NEP-518). MetaMask only asks once: the page signs each request with the browser key and sends the wallet's signature along, so a run or a revive never opens MetaMask. The code is in [`src/lib/player.ts`](src/lib/player.ts) and [`src/lib/connect-wallet.svelte`](src/lib/connect-wallet.svelte).

The game server sends every transaction and pays the fees. The economy contract lets the `server` account record runs, buy gems, move a browser account's earnings to its wallet and withdraw to a player's own wallet, so players never sign a NEAR transaction. Before relaying, the server checks the browser key's signature and, when the request names a wallet, that the wallet itself signed the link. A request that names no wallet acts on the browser account. A signed request expires after a minute and can be used once.

| Endpoint             | What it does                                                                                                                                                                                                                                                                                                                                                                                             |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GET /api/state`     | Round, pool, time left and gem price. With `?player=` (a wallet or a browser account), also that player's best floor, balance and gems                                                                                                                                                                                                                                                                   |
| `POST /api/link`     | Signed, after the wallet signs the link. Calls `merge_player` to move anything the browser account earned before (balance, gems, this round's best floor) to the wallet; skipped when there is nothing to move                                                                                                                                                                                           |
| `POST /api/withdraw` | Signed. Calls `withdraw_for` to send the wallet's whole balance to the wallet, so it shows in MetaMask and on NearBlocks. The contract registers the wallet on the token first (0.00125 NEAR from the economy account)                                                                                                                                                                                   |
| `GET /api/run/start` | A run ticket: the server's signed start time for `?player=`. The page gets one on load and sends it with each run                                                                                                                                                                                                                                                                                        |
| `POST /api/run`      | Signed, with the run ticket. Rejects a floor that can't be reached in the run's time (more than 1 floor a second by the page's clock, runs up to an hour) or since the ticket (more than 1 floor every 2 seconds by the server's clock). A floor above 100 counts as 100. Pays out the round if it has ended, then calls `record_run` when the floor is at least the minimum and beats the player's best |
| `POST /api/revive`   | Signed, with the gem count the page saw, so a copied request stops working after one purchase. Calls `buy_gem`; the page then tells the game to continue the run                                                                                                                                                                                                                                         |
| `POST /api/settle`   | Pays out the round if it has ended. Each open page calls it at a random moment in the 10 seconds after the countdown ends, so winnings show without waiting for someone's next run                                                                                                                                                                                                                       |

When a run ends, the game sends `pc:gameover` with an `id`, the floor, gems and duration (game PR #1), plus the player's saved gems and what a gem revive costs (50, game PR #3). The page saves the run and offers **Revive for N $PCMB**, **Revive for 50 gems** or **End run**, then replies `pc:revive`, `pc:revive-gems` or `pc:end` with the same `id`. A gem revive is paid from the gems saved in the game, so it never reaches the server; the "revives" count in the top bar is the $PCMB ones.

Known limits:

- A script that waits long enough can still send a made-up but possible run. The server's clock makes it wait, at least 2 seconds a floor, and the cap of floor 100 means one fake run can at best tie the top players, not take the whole pool. Nothing stops one person from playing with several browsers or wallets either, and each saved run costs the server gas.
- Copies of one request sent at the same moment to different server instances can both get through; a shared store would close that.
- Winnings stay in the economy contract as an in-game balance until the player clicks **Withdraw**, so revives can be paid without a popup.
- Until a wallet is connected, a balance belongs to the key in the browser's `localStorage`. Clearing site data, or switching browser or device, loses it. Connecting a wallet moves it to the wallet for good.
- The link signature is saved per browser. A new browser or device asks MetaMask to sign once more; the balance stays with the wallet. The signature does not expire and can't be revoked; with it and the browser key someone could spend the wallet's game balance on gems, but not withdraw it anywhere else.
- The game server's key can move any player's game balance (`merge_player`), so it has custody of balances that have not been withdrawn. It is a function-call key limited to the economy contract, kept only in the server's environment.
- Each first withdrawal to a new wallet costs the economy account 0.00125 NEAR for token storage. Keep it topped up.

### Mainnet

The same contracts also run on NEAR mainnet, with daily rounds:

| Account                                                                            | Role                                                   |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------ |
| [`pcmb.pixelclimb.near`](https://nearblocks.io/token/pcmb.pixelclimb.near)         | $PCMB token, 5,000,000 supply                          |
| [`economy.pixelclimb.near`](https://nearblocks.io/address/economy.pixelclimb.near) | Economy: 1,440-minute rounds, 1,000 $PCMB pool a round |
| `server.pixelclimb.near`                                                           | Game server (function-call key for the economy only)   |
| `pixelclimb.near`                                                                  | Owner                                                  |

The site picks its network with the `NEAR_NETWORK` environment variable (`testnet` by default, or `mainnet`). It sets the economy contract, the RPC, the MetaMask network (NEAR Protocol, chain 397, on mainnet) and the explorer links. A mainnet deployment needs `NEAR_NETWORK=mainnet` and the mainnet server account and key; see [`.env.example`](.env.example).

### Rebuilding the game

`static/game/` is built from the Solar2D project in the game repo, `hsjtechnologies/pixel-climb`, which is private, with:

```sh
scripts/game-web/build.sh /path/to/pixel-climb
```

It needs Docker, `gh`, `unzip`, `zip` and Node, and works on Linux. Solar2D's command-line builder can't target HTML5, so the script builds the game for Linux with the last Solar2D release that has a Linux builder (2024.3711, run in a container with no network), then packs the compiled game into Solar2D's HTML5 template ([`pack.mjs`](scripts/game-web/pack.mjs), which follows Solar2D's own `webPackageApp.lua`). On the way it:

- marks the game's Android/iOS plugins (Google Play Games, review popup, Twitter) as mobile-only, so the builder skips them;
- replaces the template's `alert()` calls, which would block the whole play page from inside the iframe, with a message in the game frame;
- patches two engine bugs: text with a zero line height, and a resize loop at some screen scales (a laptop at 125% to 180%) that left the game white after the intro.

### Game server setup

The server needs its own account with a key that can only call the economy contract, so a leaked key can't move NEAR. Run these once with the owner's key (near-cli-rs, legacy keychain):

```sh
OWNER=pixelclimb.testnet
ECONOMY=economy.$OWNER
SERVER=server.$OWNER

near account create-account fund-myself $SERVER '1 NEAR' autogenerate-new-keypair save-to-legacy-keychain sign-as $OWNER network-config testnet sign-with-legacy-keychain send
near account add-key $SERVER grant-function-call-access --allowance unlimited --contract-account-id $ECONOMY --function-names 'record_run, buy_gem, settle_round, merge_player, withdraw_for' autogenerate-new-keypair save-to-legacy-keychain network-config testnet sign-with-legacy-keychain send
near contract call-function as-transaction $ECONOMY set_server json-args "{\"server\":\"$SERVER\"}" prepaid-gas '30.0 Tgas' attached-deposit '0 NEAR' sign-as $OWNER network-config testnet sign-with-legacy-keychain send
```

The second command prints the path of the new key file. Its `private_key` goes in `NEAR_SERVER_KEY`, and `$SERVER` in `NEAR_SERVER_ACCOUNT`, as Vercel environment variables. See [`.env.example`](.env.example). The 1 NEAR pays for gas; top it up when it runs low.

## Try it (judges)

### On the live site

Open **https://play.pixelclimb.xyz** in a desktop browser (Chrome, Chromium or Firefox; Brave's built-in wallet can block the game). Steps 4 to 8 need [MetaMask](https://metamask.io).

1. The bar above the game shows the round, the prize pool and the time left. Space, the up arrow or W jumps.
2. Play past floor 10, with no wallet. The game-over panel says the run is saved, and when the round ends your share shows in the bar above the game.
3. With a balance, crash again and click **Revive for N $PCMB**. The run continues, and the next revive costs 5% more.
4. Click **Connect Wallet** and approve the account request, then **Add network** / **Switch network**, then sign the message that lets this browser play for your wallet. MetaMask now shows _NEAR Protocol Testnet_, the nav shows your `0x…` address, and the balance you earned is now the wallet's.
5. Click **Withdraw** to send it to your wallet; it then shows in MetaMask and on NearBlocks.
6. Click the address to open the account on testnet.nearblocks.io.
7. Reload the page. You stay connected, and MetaMask doesn't ask again.
8. In MetaMask, switch to any other network; the app disconnects. Click **Disconnect** to revoke the site's access.

To check the contracts without a browser or wallet, run `node scripts/verify.mjs` (Node 22, no install).

### Locally

Requirements: Node 22.22.2 or newer and [pnpm](https://pnpm.io).

```sh
pnpm install
cp .env.example .env   # fill in NEAR_SERVER_ACCOUNT and NEAR_SERVER_KEY to save runs and buy revives
pnpm dev
```

Open the URL Vite prints (usually http://localhost:5173) and follow the steps above. Without the server variables the page still shows the round, but runs and revives return 503.

Checks: `pnpm check` (types), `pnpm lint` and `pnpm test` (request signing, the wallet link and the run check).

## Project layout

- `src/routes/+page.svelte`: the 2:3 game page, wallet nav, round and balance bar, and the game-over panel
- `src/routes/api/`: the game server endpoints (`state`, `run`, `revive`, `settle`, `link`, `withdraw`)
- `src/lib/economy.ts`: shared by page and server: contract types, the run check and request signing
- `src/lib/player.ts`: the browser key, the wallet link, and signed requests
- `src/lib/server/near.ts`: contract reads and server-signed calls
- `src/env.ts`: environment variables
- `src/lib/connect-wallet.svelte`: MetaMask to NEAR testnet (chain 398)
- `static/game/`: the game's Solar2D HTML5 build (compiled output, served as-is)
- `scripts/game-web/`: builds `static/game/` from the pixel-climb repo
- `static/fonts/`: the game's pixel font, used by the game-over panel
- `contracts/pcmb-token/`: the $PCMB token contract (Rust, NEP-141)
- `contracts/economy/`: the economy contract (Rust): prize pool, runs and gem shop

## Built during the hackathon

- Baseline: tag [`pre-hackathon`](../../tree/pre-hackathon), the empty SvelteKit template.
- Submission: tag `hackathon-submission`, added on the final commit.
- Everything built in the hackathon: [`pre-hackathon...main`](../../compare/pre-hackathon...main).

**Pre-existing work:** the Pixel Climb game itself (the Solar2D build in `static/game/`) existed before 1 Nov 2026. It is committed here as compiled output only. What was built during the hackathon is this web app (the play page, the wallet integration and the economy endpoints), the contracts in `contracts/`, the web build script in `scripts/game-web/`, and the game's web changes in the private game repo: the bridge to this page (game PR #1), revives with saved gems (#3) and the How to play screen (#4). Their result is the compiled build in `static/game/`.

## Credits

- [SvelteKit](https://svelte.dev) and [Svelte 5](https://svelte.dev), scaffolded with `sv` 1.3.0 (minimal template, TypeScript, Prettier, ESLint, Vercel adapter)
- [Solar2D](https://solar2d.com) HTML5 runtime for the game build
- [Silkscreen](https://kottke.org/plus/type/silkscreen/) pixel font by Jason Kottke, from the game, used in the game-over panel
- [MetaMask](https://metamask.io) and NEAR's Ethereum RPC for the wallet
- [near-api-js](https://github.com/near/near-api-js) and [FastNear](https://fastnear.com) RPC for the game server
- AI tools: parts of the web app code and this README were written with AI coding tools and reviewed by the team.

## License

[MIT](LICENSE)
