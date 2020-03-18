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
    <span> {{ lastTick }}</span>
    <el-row :gutter="10">
      <el-col :span="12">
        <el-checkbox v-model="leftContract.profitPercentAuto"
          >profitPercentAuto</el-checkbox
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
          v-model="leftContract.profitPercent"
          placeholder="profitPercent"
          readonly
        />
      </el-col>
      <el-col :span="12">
        <el-checkbox v-model="rightContract.profitPercentAuto"
          >profitPercentAuto</el-checkbox
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
          v-model="rightContract.profitPercent"
          placeholder="profitPercent"
          readonly
        />
      </el-col>
    </el-row>
  </div>
</template>

<script>
import Vue from "vue";
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
      leftContract: {
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
        profitPercent: 0
      },
      rightContract: {
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
        profitPercent: 0
      }
    };
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
          let {
            tick: { quote, epoch }
            // tick: { ask, bid, epoch, pip_size, quote, symbol }
          } = response;
          this.lastTick = `t: ${quote}`;
          this.proposal(epoch, this.leftContract, "leftContract");
          this.proposal(epoch, this.rightContract, "rightContract");
        },
        true
      );
    },
    proposal(epoch, contract, key) {
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

            this.profitPercentAutoChange(contractObject);
          }
        }
      );
    },
    profitPercentAutoChange(contract) {
      if (!contract.profitPercentAuto) return;

      let {
        profitPercentChangeTo,
        profitPercentScale,
        profitPercent,
        barrier,
        barrier2
      } = contract;

      let b1 = (parseFloat(barrier) + profitPercentScale).toFixed(4);
      let b2 = (parseFloat(barrier2) - profitPercentScale).toFixed(4);
      this.$set(contract, "barrier", (b1 >= 0 ? "+" : "") + `${b1}`);
      this.$set(contract, "barrier2", (b2 >= 0 ? "+" : "") + `${b2}`);

      if (profitPercent === 0 || profitPercent === profitPercentChangeTo) {
        this.$set(contract, "profitPercentScale", 0);
        return;
      }

      console.log("profitPercentScale", profitPercentScale);

      let percentDiff = profitPercent - profitPercentChangeTo;
      if (profitPercent > 100.0) {
        if (percentDiff > 0) {
          this.$set(contract, "profitPercentScale", -0.0001);
        } else {
          this.$set(contract, "profitPercentScale", 0.0001);
        }
      } else {
        if (percentDiff > 0) {
          this.$set(contract, "profitPercentScale", 0.0001);
        } else {
          this.$set(contract, "profitPercentScale", +0.0001);
        }
      }
    }
  },
  watch: {
    "leftContract.profitPercent": {
      handler: function(newVal, oldVal) {
        console.log("leftContract profitPercent ", newVal, oldVal);
        if (oldVal === 0) return;

        let {
          profitPercentAuto,
          profitPercentScale,
          profitPercentChangeTo
        } = this.leftContract;
        if (!profitPercentAuto) return;
        if (newVal === profitPercentChangeTo) {
          Vue.set(this.leftContract, "profitPercentScale", 0);
          return;
        }

        if (newVal > oldVal) {
          if (profitPercentScale >= 0)
            Vue.set(this.leftContract, "profitPercentScale", 0.0001);
          else Vue.set(this.leftContract, "profitPercentScale", -0.0001);
        } else if (newVal < oldVal) {
          if (profitPercentScale >= 0)
            Vue.set(this.leftContract, "profitPercentScale", -0.0001);
          else Vue.set(this.leftContract, "profitPercentScale", 0.0001);
        }
      },
      deep: true
    },
    "rightContract.profitPercent": {
      handler: function(newVal, oldVal) {
        console.log("rightContract profitPercent ", newVal, oldVal);
        if (oldVal === 0) return;

        let {
          profitPercentAuto,
          profitPercentScale,
          profitPercentChangeTo
        } = this.rightContract;
        if (!profitPercentAuto) return;
        if (newVal === profitPercentChangeTo) {
          Vue.set(this.rightContract, "profitPercentScale", 0);
          return;
        }

        if (newVal > oldVal) {
          if (profitPercentScale >= 0)
            Vue.set(this.leftContract, "profitPercentScale", -0.0001);
          else Vue.set(this.leftContract, "profitPercentScale", 0.0001);
        } else if (newVal < oldVal) {
          if (profitPercentScale >= 0)
            Vue.set(this.leftContract, "profitPercentScale", 0.0001);
          else Vue.set(this.leftContract, "profitPercentScale", -0.0001);
        }
      },
      deep: true
    }
  }
};
</script>

<style scoped></style>
