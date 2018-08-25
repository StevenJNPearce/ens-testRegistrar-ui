<template>
<b-container>
  <div>
    <b-progress :value="step" :max="numSteps" class="mb-3"></b-progress>
    <b-form @submit="onNextStep">
      <b-form-group  id="ensContractAddressGroup"
                    label="ENS Contract address:"
                    label-for="ensContractAddressInput">
          <b-form-input id="ensContractAddressInput"
                        type="text"
                        v-model="ensContractAddress"
                        required
                        placeholder="0x0"
                        :disabled="step > 0">
          </b-form-input>
      </b-form-group>
      <p>
        FIFS Registrar Contract: {{registrarContractAddress}}
        <br/>
      </p>
      <b-form-group id="domainNameGroup"
                    label="Domain to register:"
                    label-for="domainNameInput">
          <b-input-group append='.test'>
            <b-form-input id="domainNameInput"
                          type="text"
                          v-model="domainName"
                          required
                          placeholder="name"
                          :disabled="step > 0">
            </b-form-input>
          </b-input-group>
      </b-form-group>
      <b-button type="submit" variant="primary">Transaction</b-button>
    </b-form><br/>
  </div>
  <p v-if='step === 1'><br/>Waiting on transaction {{registrationTxHash}}</p>
  <template v-if='step >= 2'>
    <b-form @submit="onResolver">
      <b-form-group  id="resolverContractAddressGroup"
                    label="Resolver Contract address:"
                    label-for="resolverContractAddressInput">
          <b-form-input id="resolverContractAddressInput"
                        type="text"
                        v-model="resolverContractAddress"
                        required
                        placeholder="0x0"
                        :disabled="step >= 3">
          </b-form-input>
      </b-form-group>
      <b-button type="submit" variant="primary">Next Transaction</b-button>
    </b-form><br/>
    <p v-if='step === 3'><br/>Waiting on transaction {{resolverTxHash}}</p>
  </template>
  <template v-if='step >= 4'>
    <b-form @submit="onSubmit">
      <b-form-group  id="resolveToAddressGroup"
                    label="Address name resolves to:"
                    label-for="resolveToAddressGroup">
          <b-form-input id="resolveToAddressInput"
                        type="text"
                        v-model="resolveToAddress"
                        required
                        placeholder="0x0"
                        :disabled="step > 4">
          </b-form-input>
      </b-form-group>
      <b-button type="submit" variant="primary">Final Transaction</b-button>
    </b-form>
    <p v-if='step === 5'><br/>Waiting on transaction {{setAddrTxHash}}</p>
    <h3 v-if='step === 6'><br/>Done! {{domainName}}.test now resolves to {{resolveToAddress}}</h3>
  </template>
  <b-alert variant="danger"
            dismissible
            :show="showNotify"
            @dismissed="showNotify=false">
            {{notifyMessage}}
  </b-alert>
</b-container>
</template>

<script>
import { fifsRegistrarContract, ensContract, resolverContract, namehash } from './testRegistrar'
import Web3 from 'web3'
export let web3

export default {
  created () {
  if (typeof window.web3 !== 'undefined') {
    window.web3 = new Web3(window.web3.currentProvider)
  } else {
    window.web3 = new Web3(new Web3.providers.HttpProvider('http://localhost:8545'))
    this.notify('No Web3 detected!')
  }
  if(window.web3.eth.accounts[0] === null || window.web3.eth.accounts[0] === undefined) {
    this.notify('Please unlock MetaMask!')
  }
  },
  data () {
    return {
      ensContractAddress: '',
      domainName: '',
      registrarContractAddress: 'Not found',
      resolverContractAddress: '',
      registered: false,
      ens: '',
      registrar: '',
      step: 0,
      numSteps: 6,
      registrationTxHash: '',
      resolverAddress: '',
      resolverTxHash: '',
      setOwnerTxHash: '',
      setAddrTxHash: '',
      resolveToAddress: '',
      notifyMessage: '',
      showNotify: false
    }
  },
  watch: {
    ensContractAddress: async function (addr) {
      this.ens = ensContract.at(addr)
      this.registrarContractAddress = await new Promise((resolve, reject) => {
        this.ens.owner(namehash('test'), (err, res) => {
          if (err) {
            reject(err)
          } else {
          resolve(res)
          }
        })
      })
      .catch(err => this.notify(err))
    }
  },
  methods: {
    async onResolver (evt) {
      evt.preventDefault();
      this.resolverTxHash = await new Promise((resolve, reject) => {
        this.ens.setResolver(
          namehash(this.domainName + '.test'),
          this.resolverContractAddress,
          {from: window.web3.eth.accounts[0]},
          (err, res) => {
            if(err) {
              reject(err)
            } else {
              resolve(res)
            }
          }
        )
      })
      .catch(err => this.notify(err))

      this.step = 3

      this.ens.NewResolver({}, { 
        transactionHash: this.resolverTxHash, 
        fromBlock: 'latest' 
        }).watch((err, event) => {
        if(err){
          this.notify(err.message)
        } else {
          console.log(JSON.stringify(event))
          this.step = 4
        }
      })
    },
    async onNextStep (evt) {
      evt.preventDefault();
      this.registrar = fifsRegistrarContract.at(this.registrarContractAddress);
      const expiryTime = await new Promise((resolve, reject) => {
        this.registrar.expiryTimes(window.web3.sha3(this.domainName), (err, res) => {
          if (err) {
            reject(err)
          } else {
          resolve(res)
          }
        })
      })
      .catch(err => this.notify(err))

      if(new Date().getTime() < expiryTime.toNumber() * 1000) {
        this.notify('domain unavailable')
        return;
      }
      this.registrationTxHash = await new Promise((resolve, reject) => {
        this.registrar.register(
          window.web3.sha3(this.domainName),
          window.web3.eth.accounts[0],
          { from: window.web3.eth.accounts[0] },
          (err, res) => {
            if(err) {
              reject(err)
            } else {
              resolve(res)
            }
          }
        )
      })
      .catch(err => this.notify(err))

      this.step = 1

      this.ens.NewOwner({}, { 
        transactionHash: this.registrationTxHash, 
        fromBlock: 'latest' 
        }).watch((err, event) => {
        if(err){
          this.notify(err.message)
        } else {
          console.log(JSON.stringify(event))
          this.step = 2
        }
      })

    },
    async onSubmit (evt) {
      evt.preventDefault();
      const resolver = resolverContract.at(this.resolverContractAddress)
      this.setAddrTxHash = await new Promise((resolve, reject) => {
        resolver.setAddr(
          namehash(this.domainName + '.test'),
          this.resolveToAddress,
          {from: window.web3.eth.accounts[0]},
          (err, res) => {
            if(err){
              reject(err)
            } else {
              resolve(res)
              this.step = 5
            }
          }
        )
      })
      .catch(err => this.notify(err))
        resolver.AddrChanged({}, { 
        transactionHash: this.setAddrTxHash, 
        fromBlock: 'latest' 
        }).watch((err, event) => {
        if(err){
          this.notify(err.message)
        } else {
          console.log(JSON.stringify(event))
          this.step = 6
        }
      })
    },
    notify(reason) {
      this.showNotify = true
      this.notifyMessage = reason
      this.step = 0
    }
  }
}
</script>
