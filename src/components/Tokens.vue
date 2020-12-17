<template>
  <section id="features" class="section has-company-bg">
    <div class="container is-10">
      <div class="columns reverse-row-order">
        <div class="column is-5 is-offset-4">
          <h3 class="pl-3 has-text-left is-size-3">
            More Tokens 🔥🔥🔥
          </h3>
          <div class="column is-10">
            <div class="field">
              <div class="control">
                <input
                  class="input is-primary"
                  type="text"
                  v-model="token.name"
                  placeholder="Token"
                />
              </div>
            </div>
            <div class="field">
              <div class="control">
                <input
                  class="input is-primary"
                  type="number"
                  v-model="token.balanceOwner"
                  placeholder="Owner's balance"
                />
              </div>
            </div>
            <progress
              v-if="loading"
              class="progress is-small is-primary"
              max="100"
              >15%</progress
            >
            <b-button
              class="is-rounded"
              type="is-primary"
              size="is-large"
              @click="moreTokens"
              >Token 🚀</b-button
            >
          </div>
        </div>
        <div class="columns is-2">
          <div class="column has-text-right is-offset-9">
            <span class="text-grey-dark">$whoami</span>
            <div v-if="addressResponse">
              <div class="mb-2">
                <ae-address
                  class="inline-flex"
                  :value="addressResponse.result"
                  length="short"
                />
              </div>
              <div>
                <span class="text-grey-dark">💸 {{ getBalance }} AE</span>
              </div>
            </div>
          </div>
        </div>
      </div>
      <div class="columns">
        <div class="column has-text-left is-offset-4">
          <span class="text-grey-dark">$logger</span>
          <ul class="log-tokens">
            <li v-for="token in deployed" :key="token.name">
              [ name: {{ token.name }}, address: {{ token.address }} ]
            </li>
          </ul>
        </div>
      </div>
    </div>
  </section>
</template>

<script>
import { RpcAepp } from '@aeternity/aepp-sdk/es';
import Detector from '@aeternity/aepp-sdk/es/utils/aepp-wallet-communication/wallet-detector';
import BrowserWindowMessageConnection from '@aeternity/aepp-sdk/es/utils/aepp-wallet-communication/connection/browser-window-message';
import Node from '@aeternity/aepp-sdk/es/node';
import FUNGIBLE_TOKEN_CONTRACT from 'aeternity-fungible-token/FungibleTokenFull.aes';
import { AeAddress } from '@aeternity/aepp-components';

// Send wallet connection info to Aepp throug content script
const networks = {
  ae_uat: {
    NODE_URL: 'https://mainnet.aeternity.io',
    NODE_INTERNAL_URL: 'https://mainnet.aeternity.io',
    COMPILER_URL: 'https://latest.compiler.aepps.com',
  },
  ae_mainnet: {
    NODE_URL: 'https://testnet.aeternity.io',
    NODE_INTERNAL_URL: 'https://testnet.aeternity.io',
    COMPILER_URL: 'https://latest.compiler.aepps.com',
  },
};

const errorAsField = async fn => {
  try {
    return { result: await fn };
  } catch (error) {
    return { error };
  }
};

export default {
  name: 'Home',
  components: {
    AeAddress,
  },
  data() {
    return {
      token: {
        name: null,
        decimals: 18,
        symbol: null,
        balanceOwner: null,
      },
      addressResponse: null,
      walletName: null,
      balance: null,
      deployed: [],
      loading: false,
    };
  },
  computed: {
    getBalance() {
      return this.convertToAE(this.balance);
    },
  },
  methods: {
    convertToAE(balance) {
      return +(balance / 10 ** 18).toFixed(2);
    },
    logDeployed(name, deployed) {
      console.log(deployed);
      this.deployed.push({ name, address: deployed.address });
    },
    async moreTokens() {
      this.loading = true;
      this.token.symbol = this.token.name;
      const { name, decimals, symbol, balanceOwner } = this.token;
      console.log(name, decimals, symbol, balanceOwner);
      const contract = await this.client.getContractInstance(
        FUNGIBLE_TOKEN_CONTRACT,
      );
      const init = await contract.deploy([
        name,
        decimals,
        symbol,
        balanceOwner,
      ]);
      this.loading = false;
      this.logDeployed(name, init);
    },
    async disconnect() {
      await this.client.disconnectWallet();
      this.walletName = null;
      this.pub = null;
      this.balance = null;
      this.addressResponse = null;
      this.scanForWallets();
    },
    async connectToWallet(wallet) {
      await this.client.connectToWallet(await wallet.getConnection());
      this.accounts = await this.client.subscribeAddress(
        'subscribe',
        'connected',
      );
      this.pub = await this.client.address();
      this.onAccount = this.pub;
      this.balance = await this.client.getBalance(this.pub);
      this.walletName = this.client.rpcClient.info.name;
      this.addressResponse = await errorAsField(this.client.address());
      this.heightResponse = await errorAsField(this.client.height());
      this.nodeInfoResponse = await errorAsField(this.client.getNodeInfo());
      this.compilerVersionResponse = await errorAsField(
        this.client.getCompilerVersion(),
      );
    },
    async scanForWallets() {
      try {
        const handleWallets = async function({ wallets, newWallet }) {
          newWallet = newWallet || Object.values(wallets)[0];
          // if (confirm(`Do you want to connect to wallet ${newWallet.name}`)) {

          // }
          this.detector.stopScan();

          await this.connectToWallet(newWallet);
          // let addr = await this.client.askAddresses()
        };

        const scannerConnection = await BrowserWindowMessageConnection({
          connectionInfo: { id: 'spy' },
        });
        this.detector = await Detector({ connection: scannerConnection });
        this.detector.scan(handleWallets.bind(this));
      } catch (e) {
        console.log(e);
      }
    },
  },
  async created() {
    const node = await Node({
      url: networks.ae_mainnet.NODE_URL,
      internalUrl: networks.ae_mainnet.NODE_INTERNAL_URL,
    });
    this.client = await RpcAepp({
      name: 'AEPP',
      nodes: [{ name: 'ae_mainnet', instance: node }],
      compilerUrl: networks.ae_mainnet.COMPILER_URL,
      onNetworkChange(params) {
        console.log(params);
        if (this.getNetworkId() !== params.networkId)
          alert(
            `Connected network ${this.getNetworkId()} is not supported with wallet network ${
              params.networkId
            }`,
          );
      },
      onAddressChange: async addresses => {
        console.log(addresses);
        this.pub = await this.client.address();
        this.balance = await this.client.balance(this.pub).catch(e => {
          console.log(e);
          return 0;
        });
        this.addressResponse = await errorAsField(this.client.address());
      },
      onDisconnect(a) {
        console.log('disconnect');
        console.log(a);
      },
    });
    this.height = await this.client.height();

    // Start looking for wallets
    await this.scanForWallets();
  },
};
</script>

<style scoped>
.log-tokens {
  font-size: 10pt;
}
.inline-flex {
  display: inline-flex;
}
.big-title {
  font-size: 2.4375rem;
  font-weight: 700;
  line-height: 4.0625rem;
  color: #203040;
  margin: 1.5rem 0;
}
.ae-identicon {
  box-shadow: 0 0 0 2px #ff0d6a;
  border: 0.125rem solid transparent;
  width: 2.625rem !important;
  height: 2.625rem !important;
}
.ae-address {
  font-weight: bold;
}
.ae-address li {
  list-style: none;
}
.ae-button.round {
  height: 45px;
  padding: 0 3rem;
}
</style>
