<template>
  <div class="modal-card autowidth">
    <header class="modal-card-head">
      <p class="modal-card-title">Sponsor {{ countryName }}</p>
    </header>
    <section class="modal-card-body">
      <div class="columns">
        <div class="column">
          <p>To become the new sponsor of {{ countryName }}, you need to pay <strong>{{ transaction.amount | price }}</strong>.</p>
        </div>
      </div>
      <div class="columns">
        <div class="column content is-hidden-mobile">
          <h4>Pay with Scatter Desktop</h4>
          <p>Scatter Desktop allows convenient transactions securely.</p>
          <b-notification type="is-danger" has-icon :closable="false" v-if="!isScatterConnected">
            <p>Unable to detect Scatter Desktop.</p>
            <p>If you would like to pay with Scatter Desktop, please open and unlock your Scatter Desktop, then refresh this page.</p>
            <p>If you don't have one, check out: <a href="https://support.newdex.io/hc/en-us/articles/360016322611-How-to-Use-Scatter-Desktop-" target="_blank">How to use Scatter</a>.</p>
          </b-notification>
          <button :class="['button', 'is-white', 'is-rounded', 'is-outlined', { 'is-loading': isScatterLoggingIn }]"
            @click="loginScatterAsync"  :disabled="isScatterLoggingIn"
            v-if="isScatterConnected && !scatterAccount"
          >
            Login with Scatter to Continue
          </button>
          <button :class="['button', 'is-white', 'is-rounded', 'is-outlined', { 'is-loading': isScatterPaying }]"
            @click="payWithScatterAsync"
            v-if="scatterAccount" :disabled="isScatterPaying"
          >
            Pay with Scatter
          </button>
        </div>
        <div class="column content is-hidden-mobile">
          <h4>Pay with Wallet Apps</h4>
            <p>We support <a href="http://www.mathwallet.org/en/" target="_black">Math Wallet</a>,
            <a href="https://www.mytokenpocket.vip/en/" target="_black">Token Pocket</a> and
            <a href="http://meet.one/" target="_black">MEET.ONE</a>. <br>Scan QR code to pay:</p>
            <QrCode :value="walletTransferData" :options="{ size: 200 }" />
        </div>

        <div class="column content is-hidden-tablet">
          <h4>Pay with Wallet Apps</h4>
          <button :class="['button', 'is-white', 'is-rounded', 'is-outlined', { 'is-loading': isScatterPaying }]"
            @click="payWithScatterAsync" v-show="scatterAccount"
            :disabled="!scatterAccount">
            Pay in Apps
          </button>
          <button :class="['button', 'is-white', 'is-rounded', 'is-outlined', { 'is-loading': isScatterLoggingIn }]"
            @click="loginScatterAsync" v-if="isScatterConnected && !scatterAccount"
            :disabled="isScatterLoggingIn">
            Login to Continue
          </button>
        </div>
      </div>
    </section>
    <footer class="modal-card-foot">
      <button class="button is-rounded is-hidden-mobile is-primary" @click="paidWithWalletApp()">I have paid with Wallet Apps</button>
      <button class="button is-rounded is-white is-outlined" type="button" @click="$parent.close()">Close</button>
    </footer>
  </div>
</template>

<script>
import { mapState, mapActions } from 'vuex';
import SimpleWallet from '@/libs/SimpleWallet';
import API from '@/util/api';
import QrCode from '@xkeshi/vue-qrcode';

const walletHelper = new SimpleWallet('Crypto Meetups');

export default {
  name: 'SponsorPaymentModal',
  props: ['countryName', 'transaction'],
  components: {
    QrCode,
  },
  data: () => ({
    isScatterPaying: false,
  }),
  computed: {
    ...mapState(['isScatterConnected', 'scatterAccount', 'isScatterLoggingIn']),
    walletTransferData() {
      const payload = {
        to: this.transaction.to,
        amount: (this.transaction.amount / 10000).toDecimal(4),
        contract: 'eosio.token',
        symbol: 'EOS',
        precision: 4,
        dappData: this.transaction.memo,
        desc: 'Crypto Meetup - Become Country Sponsor',
        expired: Math.floor(Date.now() / 1000 + 10 * 60),
      };
      console.log(walletHelper.transfer(payload))
      return JSON.stringify(walletHelper.transfer(payload));
    },
  },
  methods: {
    ...mapActions(['loginScatterAsync', 'updateLandInfoAsync']),
    paidWithWalletApp() {
      this.updateLandInfoAsync();
      this.$toast.open({
        message: this.$t('buy_land_withApp_success'),
        type: 'is-black',
        duration: 5000,
        queue: false,
      });
      this.$parent.close();
    },
    async payWithScatterAsync() {
      this.isScatterPaying = true;
      try {
        await API.transferEOSAsync({
          from: this.scatterAccount.name,
          ...this.transaction,
        });
        this.updateLandInfoAsync();
        this.$dialog.alert({
          type: 'is-black',
          title: this.$t('buy_land_success_alert'),
          message:
            this.$t('buy_land_success_msg'),
          confirmText: this.$t('buy_land_success_comfm'),
        });
        this.$parent.close();
        this.isScatterPaying = false;
        return true;
      } catch (error) {
        console.error(error);

        let msg;
        if (error.message === undefined) {
          msg = JSON.parse(error).error.details[0].message;
        } else {
          msg = error.message;
        }

        this.$toast.open({
          message: `Transfer failed: ${msg}`,
          type: 'is-danger',
          duration: 3000,
          queue: false,
        });
      }
      this.isScatterPaying = false;
      return null;
    },
  },
};
</script>
<style scoped>
.autowidth {
  width: auto;
}
</style>
