<template>
  <div>
    <div style="position: relative">
      <el-form
        :inline="true"
        :model="form.fields"
        ref="form"
        :rules="form.rules"
      >
        <el-form-item label="AppID" prop="authorizeID">
          <el-input
            v-model="form.fields.authorizeID"
            placeholder="app_id"
            :disabled="isConnected"
            clearable
          ></el-input>
        </el-form-item>
        <el-form-item label="Token" prop="authorizeToken">
          <el-input
            v-model="form.fields.authorizeToken"
            placeholder="API token"
            :disabled="isConnected"
            clearable
          ></el-input>
        </el-form-item>
        <el-form-item>
          <el-button
            :disabled="isConnected"
            type="success"
            v-if="!isConnected"
            @click="connect"
            >Connect</el-button
          >
          <el-button type="danger" v-else>Disconnect</el-button>
        </el-form-item>
      </el-form>
      <div style="position: sticky">
        <div style="display: flex;flex-direction: row; justify-content: center">
          <el-button
            type="primary"
            @click="tickSubscribe"
            :disabled="this.tickList > 1"
            >tickSubscribe</el-button
          >
          <div v-if="this.tickList[0]">
            <div :style="tickStyle">
              {{ this.tickList[0].quote.toFixed(2) }}
            </div>
          </div>
        </div>
      </div>
      <div style="display: flex; flex-direction: row; margin: 10px 100px">
        <div
          style="flex: 1; border: 1px solid #989898;  margin: 5px;  padding: 2px"
        >
          <ul style="list-style: none;">
            <li
              v-for="(data, index) of tickList"
              :key="'tl_' + index"
              :style="{
                background:
                  data.quote
                    .toFixed(2)
                    .split('')
                    .pop() == buyForm.lastNumber
                    ? '#00dbff'
                    : '#fff'
              }"
            >
              {{ data.quote.toFixed(2) }}
            </li>
          </ul>
        </div>
        <div style="flex: 3; display: flex; flex-direction: row">
          <div
            style="flex: 1; float: left; border: 1px solid #989898;  margin: 5px;  padding: 2px"
          >
            <el-form
              ref="form-buy"
              class="buy"
              :model="buyForm"
              label-width="100px"
              label-position="right"
            >
              <el-form-item label="Last Number">
                <el-select
                  @change="proposal(tickList[0].epoch, proposalReq, () => {})"
                  style="width: 100%"
                  size="mini"
                  v-model="buyForm.lastNumber"
                  placeholder="placeholder"
                >
                  <el-option
                    v-for="item in [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]"
                    :key="item"
                    :label="item"
                    :value="item"
                  >
                  </el-option>
                </el-select>
              </el-form-item>
              <el-form-item label="Tick">
                <el-input
                  @input="proposal(tickList[0].epoch, proposalReq, () => {})"
                  v-model="buyForm.duration"
                  size="mini"
                  placeholder="placeholder"
                ></el-input>
              </el-form-item>
              <el-form-item label="Start price">
                <el-input
                  @input="startPriceChanged"
                  v-model="buyForm.startPrice"
                  size="mini"
                  placeholder="placeholder"
                ></el-input>
              </el-form-item>
              <el-form-item label="Price">
                <el-input
                  :disabled="true"
                  @input="proposal(tickList[0].epoch, proposalReq, () => {})"
                  v-model="buyForm.buyPrice"
                  size="mini"
                  placeholder="placeholder"
                ></el-input>
              </el-form-item>
              <div style="margin: 5px">
                <span>Auto buy </span
                ><el-switch
                  v-model="buyForm.autoBuy"
                  :active-value="true"
                  :inactive-value="false"
                  active-color="#13ce66"
                  inactive-color="#ff4949"
                ></el-switch>
              </div>
              <el-button
                size="mini"
                style="margin: 10px"
                type="primary"
                @click="buyClick"
                >sotib olish</el-button
              >
            </el-form>
          </div>
          <div
            style=" flex: 1; width: 200px; border: 1px solid #989898;  margin: 5px;  padding: 2px"
          >
            <div v-if="proposalData">
              <div>Berish: {{ proposalData["ask_price"] }}</div>
              <div>Qaytish: {{ proposalData["payout"] }}</div>
            </div>
            <div>
              <div><span>yutuq</span> +{{ plus.toFixed(2) }}</div>
              <div><span>yutqazish</span> -{{ minus.toFixed(2) }}</div>
              <div><span>jami</span> {{ (plus - minus).toFixed(2) }}</div>
            </div>
          </div>
        </div>
        <div
          style="flex: 2; border: 1px solid #989898;  margin: 5px;  padding: 2px"
        >
          <div v-if="buyList.length > 0">
            <div style="list-style: none; padding: 5px; font-size: 12px">
              <div
                style="margin: 2px"
                v-for="(data, index) of buyList"
                :key="'buy_' + index"
              >
                <div
                  v-if="!data.error"
                  :style="
                    `display: flex; flex-direction: row; background-color: ${resultBgColor(
                      data
                    )}`
                  "
                >
                  <div style="flex:1;">
                    <strong>olish: {{ data.buyTick.toFixed(2) }}</strong>
                  </div>
                  <div style="flex:1">
                    <strong
                      >natija:
                      <span v-if="data.lastTick">{{
                        data.lastTick.toFixed(2)
                      }}</span></strong
                    >
                  </div>
                  <div style="flex:1">
                    <strong>kirim: </strong><span>{{ getProfit(data) }}</span>
                  </div>
                </div>
                <div
                  v-else
                  style="display: flex; flex-direction: row; background-color: #989898"
                >
                  {{ date.error.message }}
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!--<div
        v-for="(item, index) of blackList"
        :key="`blackList-${index}`"
        style="color: red;"
      >
        {{ item }}
      </div>-->
      <div
        style="position: absolute; right: 10px; top: 0; display: flex; flex-direction: column"
      >
        <div v-if="!auth">No Auth</div>
        <div v-else>
          <div style="font-size: 14px">{{ auth.email }}</div>
          <div style="font-size: 12px;font-weight: bold">
            <span v-if="auth.is_virtual === 1">V</span>
            <span>{{ auth.balance }} $</span>
          </div>
          <div style="font-size: 12px;font-weight: bold">
            <span v-if="auth.is_virtual === 1">V</span>
            <span :style="{ color: currentBalance < 0 ? 'red' : 'green' }"
              >{{ currentBalance }} $</span
            >
          </div>
        </div>
      </div>
    </div>
    <div style="position: absolute; bottom: 10px; right: 10px">
      <el-popover
        ref="popover1"
        placement="top-start"
        title="addtoken"
        width="150"
        trigger="click"
      >
        <el-input
          size="mini"
          v-model="addtoken"
          @change="addMyToken"
          placeholder="code"
        ></el-input>
        <el-button slot="reference" v-if="!isConnected" type="text" size="mini"
          >add</el-button
        >
      </el-popover>
    </div>
  </div>
</template>

<script>
// import Vue from "vue";
// import uuid from "../plugin/uuid";
export default {
  data() {
    return {
      addtoken: "",
      form: {
        fields: {
          authorizeID: "22251",
          authorizeToken: "",
          myToken: ["0HbSIO4UqqX5v8k", "XYo0Zu4aHyn5GFa", "Q5wPdGfRV9YeG4U"]
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
      buyForm: {
        lastNumber: 0,
        duration: 2,
        startPrice: 1,
        buyPrice: 1,
        martinStep: 3,
        autoBuy: false
      },
      buyList: [],
      buyCnt: 0,
      plus: 0,
      minus: 0,
      buyMartin: {
        cnt: 0,
        enable: false,
        promiseResolve: null,
        cacheBuyAmount: null,
        maxCount: 5,
        buyCount: 4
      },
      auth: null,
      sbTick: null,
      oldSbTick: null,
      tickList: [],
      activeRequests: {},
      isConnected: false,
      reqId: 0,
      ws: null,
      blackList: [{ blackList: "here" }],
      proposalData: null
    };
  },
  computed: {
    currentBalance() {
      if (!this.auth) return "";
      return (this.auth.balance + this.plus - this.minus).toFixed(2);
    },
    tickStyle() {
      let style =
        "font-size: 18px; font-weight: bold; color:white;margin:5px;padding: 5px; border-radius: 5px;";
      if (this.tickList.length > 1) {
        let { quote: currentQuote } = this.tickList[0];
        let { quote: oldQuote } = this.tickList[1];
        if (currentQuote > oldQuote) style += "background-color: #29abe2;";
        if (currentQuote < oldQuote) style += "background-color: #c03;";
        if (currentQuote === oldQuote) style += "background-color: #29abe2;";
      }
      return style;
    },
    proposalReq() {
      let { lastNumber, buyPrice, duration } = this.buyForm;
      return {
        proposal: 1,
        amount: buyPrice,
        barrier: lastNumber,
        basis: "stake",
        contract_type: "DIGITDIFF",
        currency: "USD",
        duration: duration,
        duration_unit: "t",
        symbol: "R_100"
      };
    },
    lastQuoteNumber() {
      if (this.tickList.length === 0) return -1;
      let quote = this.tickList[0].quote.toFixed(2);
      let quoteLast = quote.split("").pop();
      return parseInt(quoteLast);
    }
  },
  methods: {
    getProfit(data) {
      if (!data.lastTick) return 0;
      let barrier = parseInt(data.buyRes.echo_req.parameters.barrier);
      let result = parseInt((data.lastTick.toFixed(2) * 100).toString()) % 10;
      if (barrier === result) {
        return -1 * parseInt(this.proposalData["ask_price"]);
      } else {
        return parseInt(this.proposalData["payout"]);
      }
    },
    startPriceChanged(val) {
      if (!val) return;
      this.$set(this.buyForm, "buyPrice", val);
      this.proposal(this.tickList[0].epoch, this.proposalReq, () => {});
    },
    resultBgColor(data) {
      if (!data.lastTick) return "#fff";
      let barrier = parseInt(data.buyRes.echo_req.parameters.barrier);
      let result = parseInt(
        data.lastTick
          .toFixed(2)
          .split("")
          .pop()
      );
      if (barrier === result) {
        return "rgb(255, 184, 180)";
      } else {
        return "#82ff90";
      }
    },
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
        this.$set(this, "auth", response.authorize);
      });
    },
    tickSubscribe() {
      this.send(
        { ticks: "R_100" },
        response => {
          let {
            tick
            // tick: { ask, bid, epoch, pip_size, quote, symbol }
          } = response;
          this.buyState(tick);
          this.tickList.unshift(tick);
          if (this.tickList.length > 20) this.tickList.pop();
          if (this.proposalData === null)
            this.proposal(tick.epoch, this.proposalReq, () => {});
          // console.log(quote, epoch)
          this.autoBuy();
        },
        true
      );
    },
    autoBuy() {
      let { lastQuoteNumber } = this;
      if (lastQuoteNumber != this.buyForm.lastNumber) return;
      if (this.buyForm.autoBuy) {
        if (this.buyMartin.enable) {
          this.buyForm.buyPrice =
            parseFloat(this.buyMartin.cacheBuyAmount) *
            Math.pow(4, this.buyMartin.cnt);
          this.buyForm.autoBuy = false;
          new Promise(resolve => {
            this.buyMartin.promiseResolve = resolve;
          }).then(() => {
            this.buyForm.autoBuy = true;
          });
        }
        this.buyContract(
          this.tickList[0].epoch,
          this.tickList[0].quote,
          this.proposalReq
        );
      }
    },
    buyState(tick) {
      if (this.buyList.length > 0) {
        this.buyList.forEach(x => {
          if (x.duration !== 0) {
            x.duration -= 1;
            if (x.duration === 0) {
              if (
                this.buyMartin.enable &&
                this.buyMartin.promiseResolve !== null
              ) {
                this.buyMartin.promiseResolve();
                this.buyMartin.promiseResolve = null;
              }

              x.lastTick = tick.quote;
              let barrier = parseInt(x.buyRes.echo_req.parameters.barrier);
              let result = parseInt(
                x.lastTick
                  .toFixed(2)
                  .split("")
                  .pop()
              );
              let { buy_price, payout } = x.buyRes.buy;
              if (barrier === result) {
                if (!this.buyMartin.enable) {
                  this.buyMartin.enable = true;
                  this.buyMartin.cacheBuyAmount = this.buyForm.buyPrice;
                }
                if (this.buyMartin.enable) {
                  this.$set(
                    this.buyMartin,
                    "buyCount",
                    this.buyForm.martinStep
                  );
                  this.buyMartin.cnt++;
                }
                this.minus += buy_price;
                if (this.buyMartin.maxCount < this.buyMartin.cnt) {
                  let { martinStep } = this.buyForm;
                  this.buyForm.autoBuy = false;
                  this.buyForm.buyPrice = this.buyMartin.cacheBuyAmount;
                  this.$set(this, "buyMartin", {
                    cnt: 0,
                    enable: false,
                    cacheBuyAmount: null,
                    maxCount: 5,
                    buyCount: martinStep
                  });
                }
              } else {
                if (this.buyMartin.enable) {
                  this.buyMartin.buyCount--;
                  if (this.buyMartin.buyCount === 0) {
                    let { martinStep } = this.buyForm;
                    this.buyForm.buyPrice = this.buyMartin.cacheBuyAmount;
                    this.$set(this, "buyMartin", {
                      cnt: 0,
                      enable: false,
                      cacheBuyAmount: null,
                      maxCount: 5,
                      buyCount: martinStep
                    });
                  }
                }
                this.plus += payout - buy_price;
              }
            }
          }
        });
      }
    },
    proposal(epoch, contract) {
      let {
        proposal,
        amount,
        barrier,
        basis,
        contract_type,
        currency,
        duration,
        duration_unit,
        symbol
      } = contract;

      this.send(
        {
          proposal,
          contract_type,
          currency,
          symbol,
          amount: parseFloat(amount),
          barrier,
          basis,
          duration: parseInt(duration),
          duration_unit,
          trading_period_start: epoch
        },
        response => {
          if (response["error"]) {
            console.error(response["error"]["message"]);
            return;
          }
          if (
            !response["proposal"] ||
            !response["proposal"]["ask_price"] ||
            !response["proposal"]["payout"]
          ) {
            console.log("Error percent: ", response);
          }
          if (response["proposal"]) {
            this.$set(this, "proposalData", response["proposal"]);
          }
        }
      );
    },
    buyContract(epoch, quote, contract) {
      let {
        amount,
        barrier,
        basis,
        contract_type,
        currency,
        duration,
        duration_unit,
        symbol
      } = contract;

      let amountF = parseFloat(amount);

      this.send(
        {
          buy: "1",
          price: amountF + 10,
          parameters: {
            amount: amountF,
            barrier,
            basis,
            contract_type,
            currency,
            duration: parseInt(duration),
            duration_unit,
            product_type: "basic",
            symbol,
            trading_period_start: epoch
          }
        },
        response => {
          if (response.error) {
            this.buyForm.autoBuy = false;
            this.$notify({
              type: "error",
              title: "Sotib olish xatoligi",
              message: response.error.message,
              duration: 5000
            });
            return;
          }
          this.buyList.unshift({
            buyRes: response,
            duration,
            buyTick: quote,
            lastTick: null
          });
          if (this.buyList.length > 20) this.buyList.pop();
          console.log("buy res", response);
        }
      );
    },
    addMyToken(val) {
      if (!this.isConnected) {
        let token = "";
        if (val === "1332") token = this.form.fields.myToken[0];
        if (val === "4448") token = this.form.fields.myToken[1];
        if (val === "7992") token = this.form.fields.myToken[2];
        this.$set(this.form.fields, "authorizeToken", token);
      }
    },
    buyClick() {
      this.buyContract(
        this.tickList[0].epoch,
        this.tickList[0].quote,
        this.proposalReq
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
.buy/deep/.el-form-item {
  margin-bottom: 0;
}
</style>
