<template>
  <div id="app">
    <div class="container">
      <div class="row">

        <div class="col-12">
          <p>A <a @click="playBeep" href="javascript:;">beep</a> will warn you every time you are ready to mine.</p>
          <div class="timer-ctn">

            <form action="" @submit="startAlarm">
              <div class="form-group">
                <input type="text" class="form-control text-center" placeholder="banana.wam" v-model="account" required
                       disabled v-show="account">
              </div>
              <!-- /.form-group -->
              <button :disabled="isLoading" class="btn btn-primary" type="submit" v-show="!timerIsRunning">
                <span v-if="!isLoading">Start alarm</span>
                <span v-if="isLoading">Loading...</span>
              </button> &nbsp;
              <button @click="abortCountdown" class="btn btn-danger" type="button" v-show="timerIsRunning"
                      :disabled="isLoading">Stop alarm
              </button>
              <!-- /.btn btn-primary -->

              &nbsp;<button @click="server_mine" class="btn btn-info" type="button"
                            :disabled="!account || isMining || mine_work !== ''">Mine
            </button>
              &nbsp;<button @click="server_claim" class="btn btn-warning" type="button" v-show="mine_work"
                            :disabled="isClaiming">Claim
            </button>
              <p v-show="isMining">
                Mining...
              </p>
              <p v-if="isClaiming">Claiming... TLM</p>

              <p v-if="last_tlm">Last claim: <br>
                <span style="font-size: 40px">{{ last_tlm }} TLM</span>
              </p>
            </form>

            <br>

            <span class="alert alert-danger" v-if="showError">
              <b>Error:</b> unable to fetch account information.
            </span>
            <!-- /.alert alert-danger -->

            <div class="timer-countdown" v-show="false">
              <vue-countdown :time="time" :auto-start="false" ref="countdown" @end="timerEnded"
                             @progress="handleCountdownProgress">
                <template slot-scope="props">
                  <b>Next Mine Attempt in:</b> <br>
                  <div class="clock">
                    <span class="n">{{ zeroPad(props.hours) }}</span> : <span class="n">{{
                      zeroPad(props.minutes)
                    }}</span> : <span class="n">{{ zeroPad(props.seconds) }}</span>
                  </div>
                  <!-- /.clock -->
                </template>
              </vue-countdown>
            </div>
            <!-- /.timer-countdown -->


          </div>
          <!-- /.timer-ctn -->

        </div>
        <!-- /.col-12 -->
        <div class="col-12">

          <p v-if="bag.length">
            {{ bag.map(i => i.data.name).join(' | ') }}
          </p>
          <p class="text-center" v-if="land && land.name">
            {{ land.name }} - {{ land.data.x }} x {{ land.data.y }} <br>
            comission {{ commision }}%
          </p>
        </div>
        <!-- /.col-12 -->
      </div>
      <!-- /.row -->
    </div>
    <!-- /.container -->

  </div>
</template>

<script>
import VueCountdown from '@chenfengyuan/vue-countdown';
import Cookies from 'js-cookie';
import {ExplorerApi} from "atomicassets"
import * as waxjs from "@waxio/waxjs/dist";
import {
  RpcError,
  Serialize
} from "eosjs";
import {TextDecoder, TextEncoder} from "text-encoding";

import $ from "jquery";

window.originalFetch = require('isomorphic-fetch');
window._fetch = require('fetch-retry')(window.originalFetch);

const fetch = function (url, params) {
  if (params === undefined) {
    params = {};
  }

  url = url.replace('/wax3.api', '/wax.api').replace('/wax2.api', '/wax.api');

  params.retries = 4;
  params.retryDelay = 2000;
  params.retryOn = [500, 504, 422, 503];

  if (/push_transaction/.test(url)) {
    params.retryDelay = 5000;
  }


  // ignore localhost
  if (url.indexOf('127.0.0.1') > -1) {
    params.retries = 0;
  }

  return window._fetch(url, params);
}

window.fetch = fetch;
window.assets = {};


/* eslint-disable no-unused-vars */

const mining_account = "m.federation";
const federation_account = "federation";
const aa_api = new ExplorerApi('https://wax.api.atomicassets.io', 'atomicassets', {fetch, rateLimit: 4,});
const hyperion_endpoints = ["https://api-wax.eosarabia.net"];
const wax = new waxjs.WaxJS('https://chain.wax.io');
/* eslint-enable no-unused-vars */


const getBag = async (mining_account, account, eos_rpc, aa_api) => {
  const bag_res = await eos_rpc.get_table_rows({
    code: mining_account,
    scope: mining_account,
    table: 'bags',
    lower_bound: account,
    upper_bound: account
  });
  const bag = [];
  if (bag_res.rows.length) {
    const items_p = bag_res.rows[0].items.map(async (item_id) => {

      // assets cache

      if (window.assets[item_id]) {
        return window.assets[item_id];
      } else {
        return aa_api.getAsset(item_id).then((data) => {
          // if (data && data.schema && data.schema.schema_name && data.schema.schema_name === 'tool.worlds'){
          window.assets[item_id] = data;
          // }

          return data;
        });

      }

    });
    return await Promise.all(items_p);
  }
  return bag;
}

const getLand = async (federation_account, mining_account, account, eos_rpc, aa_api) => {
  try {
    const miner_res = await eos_rpc.get_table_rows({
      code: mining_account,
      scope: mining_account,
      table: 'miners',
      lower_bound: account,
      upper_bound: account
    });
    let land_id;
    if (miner_res.rows.length === 0) {
      return null;
    } else {
      land_id = miner_res.rows[0].current_land;
    }

    return await getLandById(federation_account, land_id, eos_rpc, aa_api);
  } catch (e) {
    console.error(`Failed to get land - ${e.message}`);
    throw e;
    // return null;
  }
}

const getLandById = async (federation_account, land_id, eos_rpc, aa_api) => {
  try {
    const land_res = await eos_rpc.get_table_rows({
      code: federation_account,
      scope: federation_account,
      table: 'landregs',
      lower_bound: land_id,
      upper_bound: land_id
    });
    let landowner = 'federation';
    if (land_res.rows.length) {
      landowner = land_res.rows[0].owner;
    }

    if (!landowner) {
      throw new Error(`Land owner not found for land id ${land_id}`);
    }

    const land_asset = await aa_api.getAsset(land_id);

    // CACHE ASSET
    // let land_asset = null;
    // if (window.assets[land_id]) {
    //   land_asset = window.assets[land_id];
    // } else {
    //   land_asset = await aa_api.getAsset(land_id);
    //   window.assets[land_id] = land_asset;
    // }

    // const land_data = await land_asset.toObject();

    // land_asset.data.planet = intToName(land_asset.data.planet);

    // make sure these attributes are present
    land_asset.data.img = land_asset.data.img || '';
    land_asset.owner = land_asset.owner || landowner;

    let comission = land_asset.mutable_data.commission / 100;

    $(window).trigger('comission', comission);
    $(window).trigger('land', land_asset);


    return land_asset;
  } catch (e) {
    console.log(`Error in getLandById ${e.message}`);
    return null;
  }
}

const getLandMiningParams = (land) => {
  const mining_params = {
    delay: 0,
    difficulty: 0,
    ease: 0
  };

  mining_params.delay += land.data.delay;
  mining_params.difficulty += land.data.difficulty;
  mining_params.ease += land.data.ease;

  return mining_params;
};

const getBagMiningParams = (bag) => {
  const mining_params = {
    delay: 0,
    difficulty: 0,
    ease: 0
  };

  let min_delay = 65535;

  for (let b = 0; b < bag.length; b++) {
    if (bag[b].data.delay < min_delay) {
      min_delay = bag[b].data.delay;
    }
    mining_params.delay += bag[b].data.delay;
    mining_params.difficulty += bag[b].data.difficulty;
    mining_params.ease += bag[b].data.ease / 10;
  }

  if (bag.length === 2) {
    mining_params.delay -= parseInt(min_delay / 2);
  } else if (bag.length === 3) {
    mining_params.delay -= min_delay;
  }

  return mining_params;
};

const getNextMineDelay = async (mining_account, account, params, eos_rpc) => {
  const state_res = await eos_rpc.get_table_rows({
    code: mining_account,
    scope: mining_account,
    table: 'miners',
    lower_bound: account,
    upper_bound: account
  });

  let ms_until_mine = -1;
  const now = new Date().getTime();
  // console.log(`Delay = ${params.delay}`);

  if (state_res.rows.length && state_res.rows[0].last_mine_tx !== '0000000000000000000000000000000000000000000000000000000000000000') {
    // console.log(`Last mine was at ${state_res.rows[0].last_mine}, now is ${new Date()}`);
    const last_mine_ms = Date.parse(state_res.rows[0].last_mine + '.000Z');
    ms_until_mine = last_mine_ms + (params.delay * 1000) - now;

    if (ms_until_mine < 0) {
      ms_until_mine = 0;
    }
  }
  // console.log(`ms until next mine ${ms_until_mine}`);

  return ms_until_mine;
};

const getMineDelay = async function (account) {
  try {
    const bag = await getBag(mining_account, account, wax.api.rpc, aa_api);
    $(window).trigger('bag', {items: bag});

    const land = await getLand(
        federation_account,
        mining_account,
        account,
        wax.api.rpc,
        aa_api
    );
    const params = getBagMiningParams(bag);
    const land_params = getLandMiningParams(land);
    params.delay *= land_params.delay / 10;
    params.difficulty += land_params.difficulty;
    var minedelay = await getNextMineDelay(
        mining_account,
        account,
        params,
        wax.api.rpc
    );
    return minedelay;
  } catch (error) {
    console.log(error);
    throw error;
    // return error;
  }
};

window.myaudio = new Audio("https://www.tradingview.com/static/bundles/calling.a4d0d2424e7055609cc4b3e053d83cb7.mp3");

////////////////////////////// MINE

/* eslint-disable no-async-promise-executor */

const getBagDifficulty = async function (account) {
  try {
    const bag = await getBag(mining_account, account, wax.api.rpc, aa_api);
    const params = getBagMiningParams(bag);
    return params.difficulty;
  } catch (error) {
    return error;
  }
};

const getLandDifficulty = async function (account) {
  try {
    const land = await getLand(
        federation_account,
        mining_account,
        account,
        wax.api.rpc,
        aa_api
    );
    const params = getLandMiningParams(land);
    return params.difficulty;
  } catch (error) {
    return error;
  }
};

const lastMineTx = async (mining_account, account, eos_rpc) => {
  const state_res = await eos_rpc.get_table_rows({
    code: mining_account,
    scope: mining_account,
    table: 'miners',
    lower_bound: account,
    upper_bound: account
  });
  let last_mine_tx = '0000000000000000000000000000000000000000000000000000000000000000';
  if (state_res.rows.length) {
    last_mine_tx = state_res.rows[0].last_mine_tx;
  }

  return last_mine_tx;
};
/* uint8array to / from hex strings */
const fromHexString = hexString =>
    new Uint8Array(hexString.match(/.{1,2}/g).map(byte => parseInt(byte, 16)));

const nameToArray = (name) => {
  const sb = new Serialize.SerialBuffer({
    textEncoder: new TextEncoder,
    textDecoder: new TextDecoder
  });

  sb.pushName(name);

  return sb.array;
}

window.doWorkWorker = async function (mining_params) {
  console.log('mining_params', mining_params)

  const _doWorkWorker = `async function (_message) {
    let getRand = () => {
      let arr = new Uint8Array(8);
      for (let i = 0; i < 8; i++) {
        let rand = Math.floor(Math.random() * 255);
        arr[i] = rand;
      }
      return arr;
    };

    let toHex = (buffer) => {
      return [...new Uint8Array(buffer)]
          .map(b => b.toString(16).padStart(2, "0"))
          .join("");
    };

    // console.log('in worker')
    /* eslint-disable no-unused-vars */
    let {mining_account, account, account_str, difficulty, last_mine_tx, last_mine_arr, sb} = _message.data;

    account = account.slice(0, 8);

    let is_wam = account_str.substr(-4) === '.wam';

    let good = false, itr = 0, rand = 0, hash, hex_digest, rand_arr, last;

    /* eslint-disable no-unused-vars */

    console.log("Performing work with difficulty "+difficulty+", last tx is "+last_mine_tx+"...");
    if (is_wam) {
      console.log("Using WAM account");
    }

    let start = (new Date()).getTime();

    while (!good) {
      rand_arr = getRand();

      // console.log('combining', account, last_mine_arr, rand_arr);
      let combined = new Uint8Array(account.length + last_mine_arr.length + rand_arr.length);
      combined.set(account);
      combined.set(last_mine_arr, account.length);
      combined.set(rand_arr, account.length + last_mine_arr.length);

      // hash = crypto.createHash("sha256");
      // hash.update(combined.slice(0, 24));
      // hex_digest = hash.digest('hex');
      // console.log('combined slice', combined.slice(0, 24))
      hash = await crypto.subtle.digest('SHA-256', combined.slice(0, 24));
      // console.log(hash);
      hex_digest = toHex(hash);
      // console.log(hex_digest);
      if (is_wam) {
        // easier for .wam accounts
        good = hex_digest.substr(0, 4) === '0000';
      } else {
        good = hex_digest.substr(0, 6) === '000000';
      }

      if (good) {
        if (is_wam) {
          last = parseInt(hex_digest.substr(4, 1), 16);
        } else {
          last = parseInt(hex_digest.substr(6, 1), 16);
        }
        good &= (last <= difficulty);
        // console.log(hex_digest, good);
      }
      itr++;

      if (itr % 1000000 === 0) {
        console.log("Still mining - tried "+itr+" iterations");
      }

      if (!good) {
        hash = null;
      }

    }
    let end = (new Date()).getTime();

    // console.log(sb.array.slice(0, 20));
    // let rand_str = Buffer.from(sb.array.slice(16, 24)).toString('hex');
    let rand_str = toHex(rand_arr);

    console.log("Found hash in "+itr+" iterations with "+account+" "+rand_str+", last = "+last+", hex_digest "+hex_digest+" taking "+((end - start) / 1000)+"s")
    let mine_work = {account: account_str, rand_str, hex_digest};

    this.postMessage(mine_work);

    return mine_work;
  }`

  // console.log(_doWorkWorker.toString());

  mining_params.last_mine_tx = mining_params.last_mine_tx.substr(0, 16); // only first 8 bytes of txid
  mining_params.last_mine_arr = fromHexString(mining_params.last_mine_tx);

  const sb = new Serialize.SerialBuffer({
    textEncoder: new TextEncoder,
    textDecoder: new TextDecoder
  });
  mining_params.sb = sb;

  mining_params.account_str = mining_params.account;
  mining_params.account = nameToArray(mining_params.account);

  let b = new Blob(["onmessage =" + _doWorkWorker.toString()], {type: "text/javascript"});
  let worker = new Worker(URL.createObjectURL(b));
  worker.postMessage(mining_params);
  return await new Promise(resolve => worker.onmessage = e => resolve(e.data));
};

window.background_mine = async (account) => {
  return new Promise(async (resolve) => {
    const bagDifficulty = await getBagDifficulty(account);
    const landDifficulty = await getLandDifficulty(account);
    const difficulty = bagDifficulty + landDifficulty;
    console.log('difficulty', difficulty);

    console.log('start doWork = ' + Date.now());
    const last_mine_tx = await lastMineTx(mining_account, account, wax.api.rpc);

    window.doWorkWorker({mining_account, account, difficulty, last_mine_tx}).then(
        (mine_work) => {
          console.log('end doWork = ' + Date.now());
          resolve(mine_work);
        }
    );
  });
};

async function server_mine(account) {
  try {
    return window.background_mine(account).then((mine_work) => {
      return mine_work;
    });
  } catch (error) {
    console.error(error.message);
  }
}

const claim = (mining_account, account, account_permission, mine_data, hyperion_endpoints, eos_api) => {

  return new Promise(async (resolve, reject) => {
    try {
      const actions = [{
        account: mining_account,
        name: 'mine',
        authorization: [{
          actor: account,
          permission: account_permission,
        }],
        data: mine_data
      }];

      // COSiGN
      if (/cosign/.test(location.href)) {
        const res = await sendTransaction(actions)
        console.log(res)
        resolve(res.transaction_id);
      }else{
        const res = await eos_api.transact({
          actions
        }, {
          blocksBehind: 3,
          expireSeconds: 90, // todo maybe 60
          // expireSeconds: 60 * 5,
        });
        console.log(res.transaction_id)
        resolve(res.transaction_id);
      }

    } catch (e) {
      console.log(`Failed to push mine results ${e.message}`);
      reject(e);
    }
  });
}

async function server_claim2(mining_account1, account, account_permission, mine_data1, hyperion_endpoints) {
  console.log(mining_account1);
  console.log(account);
  console.log(account_permission);
  console.log(mine_data1);
  console.log(hyperion_endpoints);
  // try {
  // var mine_work = JSON.parse(mine_data1);
  var mine_work = mine_data1;
  const mine_data = {
    miner: mine_work.account,
    nonce: mine_work.rand_str,
  };
  console.log("Claiming Now");
  const claimData = await claim(mining_account, account, 'active', mine_data, hyperion_endpoints, wax.api);
  console.log("Claim Data " + claimData);
  return claimData;

}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

const getBountyFromTx = async (transaction_id, miner) => { // hyperion_endpoints arg removed
                                                           // temporary fix
  await sleep(4000);

  return new Promise(async (resolve) => {
    for (let i = 0; i < 30; i++) {
      for (let h = 0; h < hyperion_endpoints.length; h++) {
        const hyp = hyperion_endpoints[h];
        if (hyp != 'https://wax.eosusa.news') {
          try {
            const url = `${hyp}/v2/history/get_transaction?id=${transaction_id}`
            const t_res = await fetch(url);
            const t_json = await t_res.json();
            if (t_json.executed) {
              let amount = 0
              const amounts = t_json.actions.filter(a => a.act.name === 'transfer').map(a => a.act).filter(a => a.data.to === miner).map(a => a.data.quantity)
              amounts.forEach(a => amount += parseFloat(a))
              if (amount > 0) {
                console.log(`${amount.toFixed(4)} TLM`)
                resolve(amount.toFixed(4));
                return
              }
            }
          } catch (e) {
            console.log(e.message)
          }
        }

        await sleep(1000);
      }

      await sleep(2000);
    }

    resolve('UNKNOWN');
  });
}
let metaRefreshCount = 0;

let metaRefresh = ()=>{
  metaRefreshCount +=1;
  if (metaRefreshCount > 60){
    clearInterval(metaRefreshInterval);
    location.reload();
  }
}
let metaRefreshInterval = setInterval(metaRefresh, 1000);



// COISGN START

// this also affects the serialization
const transactionHeader = {
  blocksBehind: 3,
  expireSeconds: 60,
}

const sendTransaction = async actions => {
  const tx = {
    actions,
  }

  let pushTransactionArgs;

  let serverTransactionPushArgs;

  try {
    serverTransactionPushArgs = await serverSign(tx, transactionHeader)
  } catch (error) {
    console.error(`Error when requesting server signature: `, error.message)
  }

  if (serverTransactionPushArgs) {
    // just to initialize the ABIs and other structures on api
    // https://github.com/EOSIO/eosjs/blob/master/src/eosjs-api.ts#L214-L254
    await wax.api.transact(tx, {
      ...transactionHeader,
      sign: false,
      broadcast: false,
    })

    // fake requiredKeys to only be user's keys
    const requiredKeys = await wax.api.signatureProvider.getAvailableKeys()
    // must use server tx here because blocksBehind header might lead to different TAPOS tx header
    const serializedTx = serverTransactionPushArgs.serializedTransaction
    const signArgs = {
      chainId: wax.api.chainId,
      requiredKeys,
      serializedTransaction: serializedTx,
      abis: [],
    }
    pushTransactionArgs = await wax.api.signatureProvider.sign(signArgs)
    // add server signature
    pushTransactionArgs.signatures.unshift(
        serverTransactionPushArgs.signatures[0]
    )
  } else {
    // no server response => sign original tx
    pushTransactionArgs = await wax.api.transact(tx, {
      ...transactionHeader,
      sign: true,
      broadcast: false,
    })
  }

  return wax.api.pushSignedTransaction(pushTransactionArgs)
}

async function serverSign(
    transaction,
    txHeaders
) {
  // server cosign endpoint
  const rawResponse = await fetch('https://node-tx-signer-uhlfm7ztfq-uc.a.run.app/api/eos/sign', {
    method: 'POST',
    headers: {
      Accept: 'application/json',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ tx: transaction, txHeaders }),
  })

  const content = await rawResponse.json()
  if (content.error) throw new Error(content.error)

  const pushTransactionArgs = {
    ...content,
    serializedTransaction: Buffer.from(content.serializedTransaction, `hex`),
  }

  return pushTransactionArgs
}

// COSIGN END


export default {
  name: 'App',
  components: {
    VueCountdown
  },
  watch: {
    last_tlm(newVal){
      Cookies.set('last_tlm', newVal, {expires: 360});
    },
    mine_work(newVal) {
      if (newVal) {
        newVal = Cookies.set('pow', JSON.stringify(newVal), {expires: 1});
      }
      Cookies.set('pow', newVal, {expires: 1});
    }
  },
  data() {
    return {
      last_tlm: 0,
      isWindowOpen: false,
      neverBeeped: true,
      bag: [],
      land: {},
      commision: 0,
      secondsLeft: 0,
      isClaiming: false,
      isMining: false,
      claimed_tlm: 0,
      mine_work: '',
      version: '1.0.2',
      account: '',
      timerIsRunning: false,
      isLoading: false,
      time: 0,
      showError: false,
      isFirstRun: true,
      userStoped: false,
    }
  },
  created() {
    this.last_tlm = Cookies.get('last_tlm') || 0;
    let mine_work = Cookies.get('pow') || '';
    if (mine_work) {
      try {
        this.mine_work = JSON.parse(mine_work);
      } catch (e) {
        //
      }
    }

    $(window).on('comission', (e, comission) => {
      this.commision = comission;
    });

    $(window).on('land', (e, land) => {
      this.land = land;
    });

    $(window).on('bag', (e, bag) => {
      this.bag = bag.items;
    });

  },
  mounted() {
    $(window).on('beep', this.playBeep);

    if (/autoplay/.test(location.href)) {
      this.startAlarm();
    }

  },
  methods: {
    removeMetaRefresh(){
      clearInterval(metaRefreshInterval);
    },
    handleCountdownProgress(data) {
      this.secondsLeft = data.totalSeconds;
      document.title = `${this.zeroPad(data.hours)}:${this.zeroPad(data.minutes)}:${this.zeroPad(data.seconds)}`;
    },
    startReloadTimer() {
      clearTimeout(window.reload_timer);
      window.reload_timer = setTimeout(() => {
        if (document.title === '00:00:00') {
          location.reload();
        }
      }, 5 * 60 * 1000);

    },
    async startAlarm(e) {
      if (e) {
        e.preventDefault();
      }
      this.isLoading = true;
      this.showError = false;
      this.userStoped = false;

      try {
        if (!this.account) {
          this.account = await wax.login();

        }

        let secondsLeft = await getMineDelay(this.account);

        console.log(`mine delay: ${secondsLeft}`);

        if (this.isFirstRun) {
          this.isFirstRun = false;
          secondsLeft += 10; // add 10 seconds to first run
          // automine
          // this.server_mine();
        }

        if (secondsLeft >= 0) {

            this.time = secondsLeft;
            setTimeout(() => {
              this.$refs.countdown && this.$refs.countdown.start();
              this.timerIsRunning = true;
              console.log('startAlarm: countdown started');
              this.removeMetaRefresh();

            }, 200);

        } else {
          console.log('startAlarm: secondsLeft <= 0');

        }

        this.isLoading = false;

      } catch (e) {

        let $this = this;
        this.isLoading = false;
        console.error(e);
        console.log('startAlarm: starting again in 5 minutes');
        window.failure_timeout  = setTimeout(async () => {
          await $this.startAlarm();
        }, 5 * 60 * 1000);

      }

      // })();
    },
    async server_claim() {
      if (this.isWindowOpen || this.isClaiming) {
        console.log('server_claim: stop point');
        return;
      }

      this.startReloadTimer();

      if (!this.isMining && !this.mine_work){
        await this.server_mine();
      }

      let $this = this;
      if (this.mine_work) {
        this.isWindowOpen = true;
        try {
          console.log('server_claim: start');
          await server_claim2(null, this.account, null, this.mine_work, hyperion_endpoints).then(async (txId) => {

            console.log('server_claim2: then txId');

            if (txId !== undefined) {

              this.isClaiming = true;

              await getBountyFromTx(txId, this.account).then((tlm) => {
                this.claimed_tlm = tlm;
                this.last_tlm = tlm;
                this.mine_work = '';
                this.isClaiming = false;

                // auto mine
                console.log('server_claim2: starting alarm again');
                location.reload(); // TODO FIX

                setTimeout(() => {
                  $this.startAlarm();
                }, 2 * 1000);
              });

            } else {
              console.log('txId undefined, waiting 5 minutes to startAlarm');
              this.userStoped = true;

              window.failure_timeout  = setTimeout(() => {
                location.reload(); // TODO FIX
                $this.startAlarm();
              }, 5 * 60 * 1000);
            }

            this.isWindowOpen = false;
            this.isClaiming = false;

          });


        } catch (e) {
          console.error('server_claim failed');
          if (e instanceof RpcError){
            console.log(JSON.stringify(e.json, null, 2));
          }

          if (e.message && e.message.length && /but does not have signatures for it under a provided delay/.test(e.message)){
           location.reload()
          }

          console.log(e.message);
          console.log(e);
          this.isClaiming = false;
          this.isWindowOpen = false;
          this.userStoped = true;

          console.log('5 minutes FAILURE delay');
          window.failure_timeout = setTimeout(() => {
            location.reload(); // TODO FIX
            $this.startAlarm();
          }, 5 * 60 * 1000);
        }
      } else {

        console.error('server_claim: NO mine_work, trying again in 5 minutes');
        window.failure_timeout = setTimeout(() => {
          location.reload(); // TODO FIX
          $this.startAlarm();
        }, 5 * 60 * 1000);
      }
    },
    async server_mine() {
      if (this.account) {
        this.isMining = true;
        this.claimed_tlm = 0;

        try {

          await server_mine(this.account).then((result) => {
            // console.log(result);
            this.mine_work = result;
            this.isMining = false;
          });
        }catch (e){

          this.isMining = false;
        }

      }
    },

    timerEnded() {
      let $this = this;

      this.timerIsRunning = false;
      this.playBeep();

      setTimeout(async () => {
        await $this.server_claim();
      }, 2000);

    },
    playBeep() {
      try{
        window.myaudio.play();
      }catch (e){
        //
      }

    },
    zeroPad(n) {
      n = n + '';
      if (n.length < 2) {
        n = '0' + n;
      }
      return n;
    },
    abortCountdown() {
      this.$refs.countdown && this.$refs.countdown.abort();
      this.time = 0;
      this.timerIsRunning = false;
      this.userStoped = true;
      // document.title = 'Mining Alarm | Alien Worlds Tools';
      clearTimeout(window.reload_timer);
      clearTimeout(window.failure_timeout);
    }
  }
}
</script>

<style lang="scss">

/* Generated by Font Squirrel (https://www.fontsquirrel.com) on May 10, 2016 */
@font-face {
  font-family: 'nasalizationregular';
  src: url('./assets/nasalization-webfont.woff2') format('woff2'),
  url('./assets/nasalization-webfont.woff') format('woff');
  font-weight: normal;
  font-style: normal;

}

body {
  background: #343a40;
  color: white;
}

.timer-countdown {
  font-family: nasalizationregular, Helvetica, Arial, sans-serif;
  background: #001422;
  color: #0080ec;
  padding: 15px 0;
  font-size: 20px;
  width: 300px;
  margin: 0 auto;
  max-width: 100%;

  b {
    color: white;
    font-weight: normal;
  }

  .n {
    display: inline-block;
    width: 30px;
  }
}

.alert {
  display: inline-block;
  margin-bottom: 15px;
}

#app {
  //font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: white;
  margin-top: 15px;
}

.timer-ctn {
  .form-group {
    width: 300px;
    margin: 0 auto;
    margin-bottom: 15px;
  }
}

@media all and (max-width: 768px) {
  h1 {
    font-size: 30px;
  }
}

</style>
