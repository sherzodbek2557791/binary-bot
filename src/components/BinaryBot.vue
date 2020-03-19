<template>
  <div>
    <el-form :inline="true" :model="form.fields" ref="form" :rules="form.rules">
      <el-form-item label="AppID" prop="authorizeID">
        <el-input
          v-model="form.fields.authorizeID"
          placeholder="app_id"
          :disabled="isConnected"
          clearable
        />
      </el-form-item>
      <el-form-item label="Token" prop="authorizeToken">
        <el-input
          v-model="form.fields.authorizeToken"
          placeholder="API token"
          :disabled="isConnected"
          clearable
        />
      </el-form-item>
      <el-form-item>
        <el-button type="success" v-if="!isConnected" @click="connect"
          >Connect</el-button
        >
        <el-button type="danger" v-else>Disconnect</el-button>
      </el-form-item>
    </el-form>
    <el-button type="primary" @click="tickSubscribe">tickSubscribe</el-button>
    <el-checkbox v-model="autoBuyContract"
      >Auto buy contract every {{ buyContractTickCount }} ticks</el-checkbox
    >
    <el-input-number v-model="buyContractTickCount" :min="3" :max="1000" :disabled="autoBuyContract"></el-input-number>
    <span>
      {{ lastTickFixed }}
      <i class="recommend-barrier">{{ lastTickRecommend }}</i></span
    >
    <el-row :gutter="10">
      <el-col :span="12">
        <el-input
          v-model="leftContract.diffPercent"
          placeholder="diffPercent"
          :disabled="!leftContract.profitPercentAuto"
        />
        <el-checkbox v-model="leftContract.profitPercentAuto"
          >Profit Percent Auto</el-checkbox
        >
        <el-input
          v-model="leftContract.contract_type"
          placeholder="contract_type"
        />
        <el-input v-model="leftContract.symbol" placeholder="symbol" />
        <el-input v-model="leftContract.amount" placeholder="amount" />
        <el-input v-model="leftContract.barrier" placeholder="barrier" />
        <el-input v-model="leftContract.barrier2" placeholder="barrier2" />
        <el-input v-model="leftContract.basis" placeholder="basis" />
        <el-input v-model="leftContract.duration" placeholder="duration" />
        <el-input
          v-model="leftContract.duration_unit"
          placeholder="duration_unit"
        />
        <hr />
        <el-input
          v-model="leftContract.profitPercentText"
          placeholder="profitPercent"
          readonly
        />
        <el-input
          v-model="leftContract.profitPercentChangeTo"
          placeholder="profitPercentChangeTo"
          readonly
        />
      </el-col>
      <el-col :span="12">
        <el-input
          v-model="rightContract.diffPercent"
          placeholder="diffPercent"
          :disabled="!rightContract.profitPercentAuto"
        />
        <el-checkbox v-model="rightContract.profitPercentAuto"
          >Profit Percent Auto</el-checkbox
        >
        <el-input
          v-model="rightContract.contract_type"
          placeholder="contract_type"
        />
        <el-input v-model="rightContract.symbol" placeholder="symbol" />
        <el-input v-model="rightContract.amount" placeholder="amount" />
        <el-input v-model="rightContract.barrier" placeholder="barrier" />
        <el-input v-model="rightContract.barrier2" placeholder="barrier2" />
        <el-input v-model="rightContract.basis" placeholder="basis" />
        <el-input v-model="rightContract.duration" placeholder="duration" />
        <el-input
          v-model="rightContract.duration_unit"
          placeholder="duration_unit"
        />
        <hr />
        <el-input
          v-model="rightContract.profitPercentText"
          placeholder="profitPercent"
          readonly
        />
        <el-input
          v-model="rightContract.profitPercentChangeTo"
          placeholder="profitPercentChangeTo"
          readonly
        />
      </el-col>
    </el-row>

    <div
      v-for="(item, index) of blackList"
      :key="`blackList-${index}`"
      style="color: red;"
    >
      {{ item }}
    </div>
  </div>
</template>

<script>
import Vue from "vue";
import uuid from "../plugin/uuid";
export default {
  data() {
    return {
      form: {
        fields: {
          authorizeID: "22232",
          authorizeToken: "Q5wPdGfRV9YeG4U"
        },
        rules: {
          authorizeID: {
            required: true,
            message: "Required",
            trigger: "change"
          },
          authorizeToken: {
            required: true,
            message: "Required",
            trigger: "change"
          }
        }
      },
      activeRequests: {},
      isConnected: false,
      reqId: 0,
      ws: null,
      lastTick: null,
      lastTickRecommend: null,
      blackList: [{ blackList: "here" }],
      autoBuyContract: false,
      buyContractTick: 0,
      buyContractTickCount: 30,
      leftContract: {
        diffPercent: 0.000671039,
        profitPercentAuto: false,
        profitPercentChangeTo: 100.1,
        profitPercentScale: 0,
        contract_type: "EXPIRYMISS",
        symbol: "R_50",
        amount: 10,
        barrier: "+0.1111",
        barrier2: "-0.1111",
        basis: "stake",
        duration: 2,
        duration_unit: "m",
        profitPercent: 0,
        profitPercentText: null,
        oldProfitPercent: 0
      },
      rightContract: {
        diffPercent: 0.0006339396875,
        profitPercentAuto: false,
        profitPercentChangeTo: 100.1,
        profitPercentScale: 0,
        contract_type: "EXPIRYRANGE",
        symbol: "R_50",
        amount: 10,
        barrier: "+0.1111",
        barrier2: "-0.1111",
        basis: "stake",
        duration: 2,
        duration_unit: "m",
        profitPercent: 0,
        profitPercentText: null,
        oldProfitPercent: 0
      }
    };
  },
  computed: {
    lastTickFixed() {
      let { lastTick } = this;
      return lastTick;
    }
  },
  methods: {
    connect() {
      let { form } = this.$refs;
      if (!form) return;
      form.validate(valid => {
        if (!valid) return;
        let {
          model: { authorizeID }
        } = form;
        console.log("connect model", form.model);
        this.ws = new WebSocket(
          `wss://ws.binaryws.com/websockets/v3?app_id=${authorizeID}`
        );
        this.ws.onopen = this.onConnect;
        this.ws.onclose = this.onDisconnect;
        this.ws.onmessage = this.onMessage;
      });
    },
    onConnect() {
      console.log("onConnect", this);
      this.isConnected = true;
      this.authorize();
    },
    onDisconnect() {
      this.isConnected = false;
    },
    onMessage(msg) {
      let data = JSON.parse(msg.data);
      this.receive(data);
    },
    send(jsonObject, callback, isSubscribe) {
      if (this.ws && this.ws.readyState === 1) {
        this.reqId++;
        let sendObject = Object.assign({ req_id: this.reqId }, jsonObject);
        this.activeRequests[this.reqId] = { sendObject, callback, isSubscribe };
        this.ws.send(JSON.stringify(sendObject));
      } else {
        this.isConnected = false;
        this.ws = null;
      }
    },
    receive(jsonObject) {
      let { req_id } = jsonObject;
      if (req_id) {
        let activeRequest = this.activeRequests[req_id];
        if (activeRequest) {
          let { callback, isSubscribe } = activeRequest;
          if (callback) {
            callback(jsonObject);
            if (!isSubscribe) {
              this.activeRequests[req_id] = null;
            }
          }
        } else {
          console.log("jsonObject", jsonObject);
        }
      } else {
        console.log("jsonObject", jsonObject);
      }
    },
    authorize() {
      this.send({ authorize: this.form.fields.authorizeToken }, response => {
        console.log("response", response);
      });
    },
    tickSubscribe() {
      this.send(
        { ticks: "R_50" },
        response => {
          this.buyContractTick++;
          let {
            tick: { quote, epoch }
            // tick: { ask, bid, epoch, pip_size, quote, symbol }
          } = response;
          this.lastTick = `t: ${quote.toFixed(4)}`;
          let diffPercentLeft = parseFloat(this.leftContract.diffPercent);
          let diffPercentRight = parseFloat(this.rightContract.diffPercent);
          this.lastTickRecommend = `${(quote * diffPercentLeft).toFixed(4)}/${(
            quote * diffPercentRight
          ).toFixed(4)}`;
          let index = 0;
          let uuid_ = uuid();
          let proposalCallback = () => {
            index++;
            console.log(
              "uuid_, index, epoch",
              uuid_,
              index,
              epoch,
              `this.leftContract=${this.leftContract.profitPercent} this.rightContract=${this.rightContract.profitPercent}`
            );
            if (
              index === 2 &&
              this.autoBuyContract &&
              this.buyContractTick % this.buyContractTickCount === 0
            ) {
              this.buyContract(epoch, this.leftContract);
              this.buyContract(epoch, this.rightContract);
            }
          };
          this.proposal(
            epoch,
            this.leftContract,
            "leftContract",
            quote,
            proposalCallback
          );
          this.proposal(
            epoch,
            this.rightContract,
            "rightContract",
            quote,
            proposalCallback
          );
        },
        true
      );
    },
    proposal(epoch, contract, key, quote, proposalCallback) {
      if (contract.profitPercentAuto) {
        let diffPercent = parseFloat(contract.diffPercent);
        this.$set(contract, "barrier", "+" + (quote * diffPercent).toFixed(4));
        this.$set(contract, "barrier2", "-" + (quote * diffPercent).toFixed(4));
      }

      let {
        contract_type,
        symbol,
        amount: amountF,
        barrier,
        barrier2,
        basis,
        duration: durationI,
        duration_unit
      } = contract;

      this.send(
        {
          proposal: 1,
          contract_type,
          currency: "USD",
          symbol,
          amount: parseFloat(amountF),
          barrier,
          barrier2,
          basis,
          duration: parseInt(durationI),
          duration_unit,
          trading_period_start: epoch
        },
        response => {
          if (response["error"]) {
            console.error(response["error"]["message"]);
            return;
          }

          let contractObject = this[key];
          if (
            !response["proposal"] ||
            !response["proposal"]["ask_price"] ||
            !response["proposal"]["payout"]
          ) {
            console.log("Error percent: ", response);
            return;
          }

          if (contractObject) {
            let {
              proposal: { ask_price, payout }
            } = response;
            let profitPercent = ((payout * 100.0) / ask_price - 100.0).toFixed(
              2
            );
            Vue.set(contract, "profitPercent", profitPercent);
            Vue.set(
              contract,
              "profitPercentText",
              `${profitPercent} (${payout})`
            );

            if (payout < parseFloat(amountF))
              this.blackList.push({ response, profitPercent });
            if (proposalCallback) proposalCallback(epoch);
            // this.profitPercentAutoChange(contractObject);
          }
        }
      );
    },
    buyContract(epoch, contract) {
      let {
        contract_type,
        symbol,
        amount: amountF,
        barrier,
        barrier2,
        basis,
        duration: durationI,
        duration_unit
      } = contract;

      this.send(
        {
          buy: "1",
          price: 20,
          parameters: {
            amount: parseFloat(amountF),
            barrier,
            barrier2,
            basis,
            contract_type,
            currency: "USD",
            duration: parseInt(durationI),
            duration_unit,
            product_type: "basic",
            symbol,
            trading_period_start: epoch
          },
          req_id: 10
        },
        response => {
          console.log("response", response);
        }
      );
    }
  }
};
</script>

<style scoped>
.recommend-barrier {
  color: #989898;
  font-size: 10pt;
}
</style>
