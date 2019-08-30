<template>
  <div id="app" class="text-center">
    <div class="row">
    <div class="col-md-6 offset-md-3">
        <b-card
          title="Simple Store"
          class="mb-2 mt-5"
        >
          <b-card-text>
            <h1>
              <b-badge variant="info">
                <span v-if="pending">
                  <b-spinner />
                </span>
                <span v-else>
                  {{ value || '?'}}
                </span>
              </b-badge>
            </h1>
              <b-form-input 
                type="number" 
                v-model="newValue" 
                placeholder="New Value" 
              />
          </b-card-text>

          <b-button variant="primary" @click="setValue" :disabled="pending">
            Set Value
            <b-spinner v-if="pending" small type="grow" />
          </b-button>
        </b-card>
      </div>
    </div>
  </div>
</template>

<script>
import Web3 from 'web3'
import { ethers } from 'ethers'
import {
  Client,
  CryptoUtils,
  LocalAddress,
  LoomProvider,
  NonceTxMiddleware,
  SignedTxMiddleware
} from 'loom-js'

const CONTRACT_ADDRESS = '0x26fd9B22c4e82A86dE23BC3511D3620C49d4B581'

export default {
  data() {
    return {
      value: null,
      newValue: null,
      loomProvider: null,
      loomContract: null,
      pending: false,
    }
  },

  async mounted() {
    await this.initLoom()
    this.initContract()
    await this.getValue()
  },

  methods: {
    async initLoom() {
      const client = new Client(
        'extdev-plasma-us1',
        'wss://extdev-plasma-us1.dappchains.com/websocket',
        'wss://extdev-plasma-us1.dappchains.com/queryws'
      )
      this.loomClient = client
      const storedKey = localStorage.getItem('loomKey')
      let privKey
      if (storedKey) {
        privKey = CryptoUtils.B64ToUint8Array(storedKey)
      } else {
        privKey = CryptoUtils.generatePrivateKey()
        localStorage.setItem('loomKey', CryptoUtils.Uint8ArrayToB64(privKey))
      }
      const pubKey = CryptoUtils.publicKeyFromPrivateKey(privKey)
      this.loomWalletAddr = LocalAddress.fromPublicKey(pubKey).toString()
      // required middleware
      client.txMiddleware = [
        new NonceTxMiddleware(pubKey, client),
        new SignedTxMiddleware(privKey)
      ]
      const provider = new LoomProvider(client, privKey)
      const web3Provider = new Web3(provider)
      this.loomProvider = new ethers.providers.Web3Provider(
        web3Provider.currentProvider
      )
    },

    initContract() {
      const abi = [
        'function set(uint _value) public @50000',
        'function get() public view returns (uint)',
        'event NewValueSet(uint _value)'
      ]

      this.loomContract = new ethers.Contract(
        CONTRACT_ADDRESS,
        abi,
        this.loomProvider.getSigner()
      )

      this.loomContract.on("NewValueSet", (value) => {
        this.value = value.toString()
        this.pending = false
      })
    },

    async setValue() {
      if (!this.newValue) return
      this.pending = true
      await this.loomContract.set(parseInt(this.newValue))
      this.newValue = null
    },

    async getValue() {
      this.value = await this.loomContract.get()
    }
  }
}
</script>
