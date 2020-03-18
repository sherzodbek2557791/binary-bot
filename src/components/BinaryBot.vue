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
    <span>
      {{ lastTickFixed }}
      <i class="recommend-barrier">{{ lastTickRecommend }}</i></span
    >
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
      leftContract: {
        diffPercent: 0.000672039,
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
        oldProfitPercent: 0
      },
      rightContract: {
        diffPercent: 0.0006335396875,
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
          let {
            tick: { quote, epoch }
            // tick: { ask, bid, epoch, pip_size, quote, symbol }
          } = response;
          this.lastTick = `t: ${quote.toFixed(4)}`;
          this.lastTickRecommend = `${(
            quote * this.leftContract.diffPercent
          ).toFixed(4)}/${(quote * this.rightContract.diffPercent).toFixed(4)}`;
          this.proposal(epoch, this.leftContract, "leftContract", quote);
          this.proposal(epoch, this.rightContract, "rightContract", quote);
        },
        true
      );
    },
    proposal(epoch, contract, key, quote) {
      this.$set(
        contract,
        "barrier",
        "+" + (quote * contract.diffPercent).toFixed(4)
      );
      this.$set(
        contract,
        "barrier2",
        "-" + (quote * contract.diffPercent).toFixed(4)
      );

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

          /*console.log(
            "porp: ",
            JSON.stringify({
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
            }),
            JSON.stringify(response)
          );*/

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

            if (payout < parseFloat(amountF))
              this.blackList.push({ response, profitPercent });

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
        barrier2,
        oldProfitPercent
      } = contract;

      if (profitPercent === 0 || profitPercent === profitPercentChangeTo) {
        this.$set(contract, "profitPercentScale", 0);
        return;
      }

      let b1 = (parseFloat(barrier) + profitPercentScale).toFixed(4);
      let b2 = (parseFloat(barrier2) - profitPercentScale).toFixed(4);
      this.$set(contract, "barrier", (b1 >= 0 ? "+" : "") + `${b1}`);
      this.$set(contract, "barrier2", (b2 >= 0 ? "+" : "") + `${b2}`);

      let percentDiff = profitPercent - profitPercentChangeTo;
      if (percentDiff >= 0) {
        if (profitPercentScale === 0) {
          this.$set(contract, "profitPercentScale", 0.0001);
          return;
        }

        // go to down
        let oldCurrentPercentDiff = parseFloat(
          (oldProfitPercent - profitPercent).toFixed(4)
        );
        let oldCurrentPercentDiffDest =
          oldCurrentPercentDiff === 0
            ? 1
            : oldCurrentPercentDiff / Math.abs(oldCurrentPercentDiff);
        console.log("oldCurrentPercentDiff", oldCurrentPercentDiff);
        console.log("oldCurrentPercentDiffDest", oldCurrentPercentDiffDest);
        this.$set(
          contract,
          "profitPercentScale",
          oldCurrentPercentDiffDest * profitPercentScale
        );
      } else {
        // go to up
      }

      /*
      let scaleDest = 0.0001;*/

      /* if (75.0 < absDiff) {
        scaleDest = 0.01;
      } else if (50.0 < absDiff) {
        scaleDest = 0.001;
      } else if (0 < absDiff) {
        scaleDest = 0.0001;
      }*/

      /*  let diffCom = 1;
      if (profitPercentScale !== 0)
        diffCom = profitPercentScale / Math.abs(profitPercentScale);

      console.log("diffCom", diffCom);*/

      /*if (percentDiff >= 0) {
        console.log("if (percentDiff >= 0) {", percentDiff);
        let oldCurrentPercentDiff = oldProfitPercent - profitPercent;
        if (oldCurrentPercentDiff < 0)
          this.$set(contract, "profitPercentScale", -1 * diffCom * scaleDest);
        else this.$set(contract, "profitPercentScale", profitPercentScale);
      } else {
        console.log("} else {", percentDiff);
        let oldCurrentPercentDiff = profitPercent - oldProfitPercent;
        if (oldCurrentPercentDiff < 0)
          this.$set(contract, "profitPercentScale", -1 * diffCom * scaleDest);
        else this.$set(contract, "profitPercentScale", profitPercentScale);
      }*/

      // oldProfitPercent;
      // profitPercent;
      // profitPercentChangeTo;
      //
      // profitPercentScale ->
      if (Math.ceil(Math.abs(oldProfitPercent - profitPercent)) > 0)
        this.$set(contract, "oldProfitPercent", profitPercent);
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
