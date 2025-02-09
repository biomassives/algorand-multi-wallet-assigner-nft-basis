<template>
  <MainLayout>
    <div class="container-fluid">
      <h1>{{ $t("swap.title") }}</h1>

      <div v-if="checkNetwork()">
        {{ $t("swap.network") }}: {{ checkNetwork() }}
      </div>
      <div v-else class="alert alert-danger">
        {{ $t("swap.network_not_supported") }}
      </div>
      <div v-if="hasSK === false" class="alert alert-danger">
        {{ $t("swap.has_sk") }}
      </div>
      <div>
        <h2>
          {{ $t("swap.swap_asset_from") }}
        </h2>
        <Dropdown
          v-model="asset"
          :options="assets"
          option-label="label"
          option-value="asset-id"
          placeholder="Source asset"
        />
        <h2>{{ $t("swap.swap_asset_to") }}</h2>
        <Dropdown
          v-model="toAsset"
          :options="assets"
          option-label="label"
          option-value="asset-id"
          placeholder="Destination asset"
        />
        <h2>{{ $t("swap.swap_amount") }}</h2>
        <input
          id="payamount"
          v-model="payamount"
          type="number"
          min="0"
          :max="maxAmount"
          :step="stepAmount"
          class="form-control"
        />
        <h2>{{ $t("swap.slippage") }}</h2>
        <input
          id="slippage"
          v-model="slippage"
          type="number"
          min="0"
          max="1"
          step="0.01"
          class="form-control"
        />
        <div>
          <div>
            <input id="useFolksCheckbox" type="checkbox" v-model="useFolks" />
            <label for="useFolksCheckbox">Use Folks router</label>
          </div>
          <div>
            <input id="useDeflexCheckbox" type="checkbox" v-model="useDeflex" />
            <label for="useDeflexCheckbox">Use Deflex</label>
          </div>
        </div>
        <div>
          <button
            class="btn my-2"
            :disabled="processingQuote"
            :class="
              allowExecuteDeflex || requiresOptIn ? 'btn-light' : 'btn-primary '
            "
            @click="clickGetQuote"
          >
            {{ $t("swap.get_quote") }}

            <span
              v-if="processingQuote"
              class="spinner-grow spinner-grow-sm"
              role="status"
              aria-hidden="true"
            />
          </button>
        </div>
        <div v-if="requiresOptIn">
          <h2>{{ $t("swap.apps_optin") }}</h2>
          <ul>
            <li v-for="app in appsToOptIn" :key="app">
              {{ app }}
            </li>
          </ul>
          <button
            class="btn my-2 btn-primary"
            :disabled="processingOptin"
            @click="clickOptInToApps"
          >
            <span
              v-if="processingOptin"
              class="spinner-grow spinner-grow-sm"
              role="status"
              aria-hidden="true"
            />
            {{ $t("swap.apps_optin_button") }}
          </button>
        </div>
        <div>
          <textarea
            v-model="txsDetails"
            disabled
            class="form-control"
            rows="5"
          />
        </div>
        <div v-if="note" class="alert alert-success my-2">
          {{ note }}
        </div>
        <div v-if="error" class="alert alert-danger my-2">
          {{ error }}
        </div>
        <div>
          <button
            v-if="useDeflex"
            class="btn my-2 mx-1"
            :disabled="!allowExecuteDeflex || processingTradeDeflex"
            :class="allowExecuteDeflex ? 'btn-primary' : 'btn-light '"
            @click="clickExecuteDeflex"
          >
            <span
              v-if="processingTradeDeflex"
              class="spinner-grow spinner-grow-sm"
              role="status"
              aria-hidden="true"
            />
            {{ $t("swap.execute_button_deflex") }}
          </button>
          <button
            v-if="useFolks"
            class="btn my-2"
            :disabled="!allowExecuteFolks || processingTradeFolks"
            :class="allowExecuteFolks ? 'btn-primary' : 'btn-light '"
            @click="clickExecuteFolks"
          >
            <span
              v-if="processingTradeFolks"
              class="spinner-grow spinner-grow-sm"
              role="status"
              aria-hidden="true"
            />
            {{ $t("swap.execute_button_folks") }}
          </button>
        </div>
      </div>
    </div>
  </MainLayout>
</template>

<script>
import MainLayout from "../layouts/Main.vue";
import { mapActions } from "vuex";
import algosdk from "algosdk";
import { FolksRouterClient, Network, SwapMode } from "@folks-router/js-sdk";

export default {
  components: {
    MainLayout,
  },
  data() {
    return {
      assets: [],
      asset: null,
      toAsset: null,
      payamount: 0,
      assetObj: {},
      deflexTxs: { groupMetadata: [] },
      folksQuote: {},
      folksTxns: [],
      txsDetails: "Select assets, quantity and request quote",
      quotes: {},
      hasSK: null,
      processingQuote: false,
      processingOptin: false,
      processingTradeDeflex: false,
      processingTradeFolks: false,
      note: "",
      error: "",
      useFolks: true,
      useDeflex: true,
      slippage: 0.1,
    };
  },
  computed: {
    account() {
      return this.$store.state.wallet.privateAccounts.find(
        (a) => a.addr == this.$route.params.account
      );
    },
    maxAmount() {
      if (!this.account) return 0;

      if (this.asset) {
        if (!this.selectedAssetFromAccount) return 0;
        return this.selectedAssetFromAccount.amount / this.decimalsPower;
      } else {
        let ret = this.account.amount / 1000000 - 0.1;
        ret = ret - this.fee;
        if (this.account["assets"] && this.account["assets"].length > 0)
          ret = ret - this.account["assets"].length * 0.1;
        return ret;
      }
    },
    stepAmount() {
      if (!this.asset) return 0.000001;
      if (!this.account) return 0.000001;
      if (!this.assetObj || this.assetObj.decimals === undefined)
        return 0.000001;
      return Math.pow(10, -1 * this.assetObj.decimals);
    },
    allowExecuteDeflex() {
      if (
        !this.deflexTxs ||
        !this.deflexTxs.txns ||
        Object.values(this.deflexTxs.groupMetadata).length <= 0
      ) {
        return false;
      }
      return !this.requiresOptIn;
      /**/
    },
    allowExecuteFolks() {
      if (this.folksTxns && this.folksQuote && this.folksTxns.length > 0) {
        return true;
      }
      return false;
      /**/
    },
    appsToOptIn() {
      const requiredAppOptIns = this.quotes?.requiredAppOptIns ?? [];
      const ret = [];
      //console.log("requiredAppOptIns", requiredAppOptIns);
      if (!this.account) return false;
      const optedInAppIds =
        "apps-local-state" in this.account
          ? this.account["apps-local-state"].map((state) => parseInt(state.id))
          : [];

      for (let i = 0; i < requiredAppOptIns.length; i++) {
        const requiredAppId = requiredAppOptIns[i];
        if (!optedInAppIds.includes(requiredAppId)) {
          ret.push(requiredAppId);
        }
      }
      return ret;
    },
    requiresOptIn() {
      return this.appsToOptIn.length > 0;
    },
  },
  watch: {
    async asset() {
      this.deflexTxs = { groupMetadata: [] };
      if (this.asset > 0) {
        this.assetObj = await this.getAsset({
          assetIndex: this.asset,
        });
      } else {
        this.assetObj = {
          "asset-id": -1,
          name: "ALGO",
          "unit-name": "Algo",
          decimals: 6,
        };
      }
      //console.log("assetObj", this.assetObj);
      this.payamount = 0;
    },
    account() {
      this.deflexTxs = { groupMetadata: [] };
      this.folksTxns = [];
      this.makeAssets();
    },
    toAsset() {
      this.deflexTxs = { groupMetadata: [] };
      this.folksTxns = [];
    },
    payamount() {
      this.deflexTxs = { groupMetadata: [] };
      this.folksTxns = [];
    },
  },
  async mounted() {
    //console.log("mounted swap");
    await this.reloadAccount();
    await this.makeAssets();
    this.prolong();

    this.asset = -1;
    this.toAsset = 452399768;
    this.payamount = 1;
    //console.log("this.assets from mount", this.assets);
  },
  methods: {
    ...mapActions({
      accountInformation: "indexer/accountInformation",
      updateAccount: "wallet/updateAccount",
      lastActiveAccount: "wallet/lastActiveAccount",
      deleteAccount: "wallet/deleteAccount",
      setTransaction: "wallet/setTransaction",
      getAsset: "indexer/getAsset",
      prolong: "wallet/prolong",
      setAccountOnline: "kmd/setAccountOnline",
      openSuccess: "toast/openSuccess",
      openError: "toast/openError",
      axiosGet: "axios/get",
      axiosPost: "axios/post",
      getSK: "wallet/getSK",
      getTransactionParams: "algod/getTransactionParams",
      sendRawTransaction: "algod/sendRawTransaction",
      waitForConfirmation: "algod/waitForConfirmation",
      signTransaction: "signer/signTransaction",
    }),

    async reloadAccount() {
      await this.accountInformation({
        addr: this.$route.params.account,
      }).then((info) => {
        if (info) {
          //console.log("info", info);
          this.updateAccount({ info });
        }
      });
      const senderSK = await this.getSK({
        addr: this.$route.params.account,
      });
      if (senderSK && senderSK.length > 0) {
        this.hasSK = true;
      } else {
        this.hasSK = false;
      }
      //console.log("senderSK", senderSK);
    },
    async makeAssets() {
      this.assets = [];
      if (this.account && this.account.amount > 0) {
        this.assets.push({
          "asset-id": "-1",
          amount: this.account.amount,
          name: "ALG",
          decimals: 6,
          "unit-name": "",
          label: "Algorand native token",
        });
      }
      if (this.account && this.account.assets) {
        for (let accountAsset of this.account.assets) {
          if (!accountAsset["asset-id"]) continue;
          const asset = await this.getAsset({
            assetIndex: accountAsset["asset-id"],
          });
          if (asset) {
            this.assets.push({
              "asset-id": accountAsset["asset-id"],
              amount: accountAsset["amount"],
              name: asset["name"],
              decimals: asset["decimals"],
              "unit-name": asset["unit-name"],
              label: `${asset["name"]} (${accountAsset["asset-id"]})`,
            });
          }
        }
      }
      //console.log("this.assets", this.assets);
    },
    async requestDeflexQuote() {
      try {
        const amount = this.payamount * 10 ** this.assetObj.decimals;
        const fromAsset = this.asset > 0 ? this.asset : 0;
        const toAsset = this.toAsset > 0 ? this.toAsset : 0;
        const chain = this.checkNetwork();
        if (!chain) {
          this.txsDetails += "\nDEFLEX: Wrong network selected";
          this.txsDetails = this.txsDetails.trim();
          this.processingQuote = false;
          return;
        }
        var algodUri = encodeURIComponent("https://mainnet-api.algonode.cloud");
        var algodToken = "";
        var algodPort = 443;
        if (chain == "testnet") {
          algodUri = encodeURIComponent("https://testnet-api.algonode.cloud");
          algodToken = "";
          algodPort = 443;
        }

        const apiKey = this.$store.state.config.deflex;
        const request = `https://api.deflex.fi/api/fetchQuote?chain=${chain}&algodUri=${algodUri}&algodToken=${algodToken}&algodPort=${algodPort}&fromASAID=${fromAsset}&toASAID=${toAsset}&atomicOnly=true&amount=${amount}&type=fixed-input&disabledProtocols=&referrerAddress=AWALLETCPHQPJGCZ6AHLIFPHWBHUEHQ7VBYJVVGQRRY4MEIGWUBKCQYP4Y&apiKey=${apiKey}`;
        const quotes = await this.axiosGet({ url: request }).catch((e) => {
          this.error = "No deflex quotes available " + e.message;
          return;
        });

        if (!quotes || !quotes.txnPayload) {
          this.error = "No deflex quotes available";
          return;
        }
        this.quotes = quotes;
        const params = JSON.stringify({
          address: this.account.addr,
          slippage: this.slippage, // 1 = 1%
          txnPayloadJSON: this.quotes.txnPayload,
          apiKey,
        });
        const config = {
          headers: {
            "Content-Type": "application/json",
          },
        };

        const txs = await this.axiosPost({
          url: "https://api.deflex.fi/api/fetchExecuteSwapTxns",
          body: params,
          config,
        }).catch((e) => {
          this.error += "No deflex quotes available " + e.message + "\n";
          return;
        });
        if (!txs) {
          this.error = "No deflex quotes available\n";
          this.processingQuote = false;
          return;
        }
        this.deflexTxs = txs;

        const ret2 = this.deflexTxs.groupMetadata
          .map((tx) => tx.labelText)
          .join(",\n");
        //console.log("ret2", ret2);
        this.txsDetails += "\nDEFLEX: " + ret2;
        this.txsDetails = this.txsDetails.trim();
      } catch (e) {
        this.openError("Error fetching quote from deflex: " + e.message);
      }
      //console.log("txs", params, this.txs, config);
    },
    getFolksClient() {
      if (this.$store.state.config.env == "mainnet-v1.0") {
        return new FolksRouterClient(Network.MAINNET);
      }
      if (this.$store.state.config.env == "mainnet") {
        return new FolksRouterClient(Network.MAINNET);
      }
      if (this.$store.state.config.env == "testnet-v1.0") {
        return new FolksRouterClient(Network.TESTNET);
      }
      if (this.$store.state.config.env == "testnet") {
        return new FolksRouterClient(Network.TESTNET);
      }
      return null;
    },
    async fetchFolksRouterQuotes() {
      try {
        const amount = this.payamount * 10 ** this.assetObj.decimals;
        const folksRouterClient = this.getFolksClient();
        if (!folksRouterClient)
          throw Error(
            "Unable to create folks router client for specified network"
          );
        const fromAsset = this.asset > 0 ? this.asset : 0;
        const toAsset = this.toAsset > 0 ? this.toAsset : 0;

        this.folksQuote = await folksRouterClient.fetchSwapQuote(
          fromAsset,
          toAsset,
          amount,
          SwapMode.FIXED_INPUT,
          15,
          10,
          "AWALLETCPHQPJGCZ6AHLIFPHWBHUEHQ7VBYJVVGQRRY4MEIGWUBKCQYP4Y"
        );
        const slippage = Math.round(this.slippage * 100);
        this.folksTxns = await folksRouterClient.prepareSwapTransactions(
          this.$route.params.account,
          slippage,
          this.folksQuote
        );
        const token = await this.getAsset({
          assetIndex: toAsset,
        });
        console.log("token", toAsset, token);
        this.txsDetails += `\nFOLKS ROUTER: Quote Amount: ${
          Number(this.folksQuote.quoteAmount) / 10 ** token.decimals
        }, Price Impact: ${
          Math.round(Number(this.folksQuote.priceImpact) * 10000) / 100
        }%, Txs fees: ${
          Number(this.folksQuote.microalgoTxnsFee) / 10 ** 6
        } Algo`;
        this.txsDetails = this.txsDetails.trim();
      } catch (e) {
        this.openError(`Error fetching quote from folks: ${e.message}`);
      }
    },
    async clickGetQuote() {
      this.prolong();
      this.note = "";
      this.error = "";
      this.processingQuote = true;
      this.txsDetails = "";
      this.deflexTxs = { groupMetadata: [] };
      this.folksTxns = [];
      var promises = [];
      if (this.useDeflex) promises.push(this.requestDeflexQuote());
      if (this.useFolks) promises.push(this.fetchFolksRouterQuotes());
      await Promise.all(promises);
      this.processingQuote = false;
    },
    checkNetwork() {
      if (this.$store.state.config.env == "mainnet-v1.0") {
        return "mainnet";
      }
      if (this.$store.state.config.env == "mainnet") {
        return "mainnet";
      }
      if (this.$store.state.config.env == "testnet-v1.0") {
        return "testnet";
      }
      if (this.$store.state.config.env == "testnet") {
        return "testnet";
      }
      return false;
    },
    async clickExecuteFolks() {
      this.prolong();
      this.processingTradeFolks = true;
      this.note = "";
      this.error = "";
      //console.log("execute clicked");
      //console.log("this.account", this.account);

      const senderSK = await this.getSK({
        addr: this.account.addr,
      });
      if (!senderSK) {
        this.processingTradeFolks = false;
        return;
      }
      //console.log("senderSK", senderSK);
      const unsignedTxns = this.folksTxns.map((txn) =>
        algosdk.decodeUnsignedTransaction(Buffer.from(txn, "base64"))
      );
      const signedTxns = unsignedTxns.map((txn) => txn.signTxn(senderSK));
      if (!signedTxns) {
        this.processingTradeFolks = false;
        return;
      }
      let tx = await this.sendRawTransaction({
        signedTxn: signedTxns,
      }).catch((e) => {
        //console.error("error doing swap", e);
        this.error = e.message;
        this.processingTradeFolks = false;
        this.openError(e.message);
        return;
      });

      let ret = "Processed in txs: ";

      if (!tx || !tx.txId) return;
      const confirmation = await this.waitForConfirmation({
        txId: tx.txId,
        timeout: 4,
      });
      if (confirmation) {
        ret += tx.txId + ", ";
      } else {
        this.processingOptin = false;
        await this.reloadAccount();
        return;
      }
      this.note = ret.trim().trim(",");
      //console.log("note", this.note, ret);
      this.processingTradeFolks = false;
    },
    async clickExecuteDeflex() {
      this.prolong();
      this.processingTradeDeflex = true;
      this.note = "";
      this.error = "";
      //console.log("execute clicked");
      //console.log("this.account", this.account);

      const senderSK = await this.getSK({
        addr: this.account.addr,
      });
      if (!senderSK) {
        this.processingTradeDeflex = false;
        return;
      }
      //console.log("senderSK", senderSK);
      const byGroup = this.deflexTxs.txns.reduce(
        (entryMap, e) =>
          entryMap.set(e.group, [...(entryMap.get(e.group) || []), e]),
        new Map()
      );
      const byGroupMap = [...byGroup].map((m) => m[1]);

      let ret = "Processed in txs: ";
      for (let group of byGroupMap) {
        //console.log("group", group);
        const signedTxns = group.map((txn) => {
          if (txn.logicSigBlob !== false) {
            return Uint8Array.from(Object.values(txn.logicSigBlob));
          } else {
            let bytes = new Uint8Array(Buffer.from(txn.data, "base64"));
            const decoded = algosdk.decodeUnsignedTransaction(bytes);
            //console.log("decoded", decoded);
            return algosdk.signTransaction(decoded, senderSK).blob;
          }
        });
        if (!signedTxns) {
          this.processingTradeDeflex = false;
          return;
        }
        console.log("signedTxns", signedTxns);
        const tx = await this.sendRawTransaction({
          signedTxn: signedTxns,
        }).catch((e) => {
          //console.error("error doing swap", e);
          this.error = e.message;
          this.processingTradeDeflex = false;
          this.openError(e.message);
          return;
        });
        if (!tx || !tx.txId) return;
        const confirmation = await this.waitForConfirmation({
          txId: tx.txId,
          timeout: 4,
        });
        if (confirmation) {
          ret += tx.txId + ", ";
        } else {
          this.processingOptin = false;
          await this.reloadAccount();
          return;
        }
        //console.log("confirmation", confirmation); /**/
      }
      this.note = ret.trim().trim(",");
      //console.log("note", this.note, ret);
      this.processingTradeDeflex = false;
    },

    async clickOptInToApps() {
      this.processingOptin = true;
      const params = await this.getTransactionParams();

      let ret = "Processed in txs: ";
      for (let app of this.appsToOptIn) {
        const appOptInTxn = algosdk.makeApplicationOptInTxn(
          this.account.addr,
          params,
          app
        );

        let signedTxn = await signTransaction({
          from: this.account.addr,
          tx: appOptInTxn,
        });
        const tx = await this.sendRawTransaction({ signedTxn }).catch((e) => {
          //console.error("error doing swap", e);
          this.error = e.message;
          this.processingTradeDeflex = false;
          return;
        });
        if (!tx || !tx.txId) {
          this.processingOptin = false;
          await this.reloadAccount();
          return;
        }
        const confirmation = await this.waitForConfirmation({
          txId: tx.txId,
          timeout: 4,
        });
        if (confirmation) {
          ret += tx.txId + ", ";
        } else {
          this.processingOptin = false;
          await this.reloadAccount();
          return;
        }

        //console.log("confirmation", confirmation);
      }

      this.note = ret.trim().trim(",");
      //console.log("note", this.note, ret);
      await this.reloadAccount();
      this.processingOptin = false;
    },
  },
};
</script>
