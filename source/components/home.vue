<!-- 
ExpressVPN Browser Extension:
Copyright 2017-2019 Express VPN International Ltd
Licensed GPL v2
-->
<template>
  <div>
    <main-header :canShowPromobar="true" />
    <div class="content">

      <svg width="365" height="123" :class="['visual-state', `visual-state-${currentInfo.state}`]">
        <g stroke="none" stroke-width="1" fill="none" fill-rule="evenodd">
          <g>
            <path d="M4.9801292898673495,0 L359.56533472842267,0 C362.3157839392382,-3.7065467329393714e-16 364.54546401829,1.4994273396804243 364.54546401829,3.3490647005667635 L364.54546401829,103.25080635319058 C303.7878866819083,116.13411403325709 243.0303093455267,122.57576804074355 182.27273200914496,122.57576804074355 C121.51515467276334,122.57576804074355 60.75757733638167,116.13411403325709 0,103.25080635319058 L0,3.3490647005667635 C-3.46548584986422e-16,1.4994273396804243 2.2296798300452827,3.7065467329393714e-16 4.9801292898673495,0 z" id="Ripple"/>
          </g>
        </g>
      </svg>

      <div class="power-button-container">
        <div class="power-button" @click="onPowerButtonClick">
          <circle-progress-bar :value="currentInfo.progress"></circle-progress-bar>
          <div v-if="['ready', 'connected', 'disconnecting'].includes(currentInfo.state)" :class="['icon', 'icon-large', 'icon-26-connect', { 'no-highlight': (currentInfo.state === 'disconnecting')}]"></div>
          <div v-else :class="['icon', 'icon-large', 'icon-23-close', { 'no-highlight': (parseInt(currentInfo.progress, 10) >= 75) }]"></div>
        </div>
      </div>
      
      <div class="engine-state">{{ localize(`main_screen_status_vpn_${currentInfo.state}_text`) }}</div>


      <div class="locations-container">
        <div :class="['current-location', 'location-box', `location-box-${currentInfo.state}`]" @click="openLocationPicker()">
          <img class="current-location-flag" :src="flagStyle" @error="this.setPlaceholder"/>
          <div class="current-location-text">
            <div class="location-box-text-type">{{ currentLocationTypeText }}</div>
            <div class="location-box-text-name">{{ selectedLocation.name }}</div>
          </div>
          <div v-if="['ready', 'connected'].includes(currentInfo.state)" class="icon icon-133-more-circle current-location-more-icon"></div>
          <div v-else class="icon icon-73-more icon-disabled"></div>
        </div>
        <transition mode="out-in">
          <hint v-if="!utils.isNullOrEmpty(hintText)" :stringKey="hintText" :iconName="hintIcon" :type="hintType" />
          
          <div v-else-if="['ready', 'connected'].includes(currentInfo.state)" class="location-buttons">
            <div class="location-box" v-for="location in visibleLocations" v-bind:key="location.id" @click="connectToLocation(location)">
                <div class="location-box-text-type">{{ localize(`main_screen_${location.type}_location_text`) }}</div>
                <div class="location-box-text-name">{{ location.name }}</div>
            </div>
          </div>
        </transition>
      </div>

      <transition name="fade" mode="out-in" :duration="{ enter: 500, leave: 0 }">
        <ratingPrompt v-if="showRatingPrompt" :discardPrompt="onDiscardRatingPrompt" :style="{ transitionDelay: transitionDelay }" />
      </transition>

      <popup :options="popupOptions"></popup>

      <div class="message-container">
        <message-item />
      </div>
    </div>
  </div>
</template>
<script>
import mixinLocation from '@/scripts/mixins/location';
import CircleProgressBar from './partials/circleProgressBar.vue';
import Hint from './partials/hint.vue';
import messageItem from './partials/messageItem.vue';
import popup from './partials/popup.vue';
import ratingPrompt from './partials/ratingPrompt.vue';
import mainHeader from './partials/mainHeader.vue';

export default {
  name: 'home',
  mixins: [mixinLocation],
  data() {
    return {
      transitionDelay: '2s',
      discardPrompt: false,
      discardHint: false,
      popupOptions: {},
      hintIcon: null,
      hintType: '',
      show4starHint: false,
      forceRatingPrompt: false,
    };
  },
  computed: {
    showRatingPrompt() {
      const requiredConnectionTime = (__IS_ALPHA__ || process.env.NODE_ENV === 'development') ? 0 : 15 * 60;
      const rating = this.currentInfo.ratingData;

      if (rating) {
        const lastDiscardDate = parseInt(localStorage.getItem('lastDiscardDate'), 10) || 0;
        const lastFailedRateDate = parseInt(localStorage.getItem('lastFailedRateDate'), 10) || 0;
        const NOW = (new Date()).getTime();

        return (
          rating.isSubscriber === true &&
          rating.isSuccessfulConnection === true &&
          rating.previousConnectionTime >= requiredConnectionTime &&
          rating.everClickedMaxRating === false &&
          this.discardPrompt === false &&
          this.daysBetween(NOW, lastDiscardDate) > 30 &&
          this.daysBetween(NOW, lastFailedRateDate) > 10 &&
          this.currentInfo.os !== 'LINUX' &&
          this.currentInfo.locale === 'en' &&
          this.currentInfo.state === 'connected'
        ) || (this.forceRatingPrompt === true);
      }
    },
    selectedLocation() {
      return this.currentInfo.selectedLocation || {};
    },
    currentLocationTypeText() {
      let stringKey = `main_screen_${this.selectedLocation.is_smart_location ? 'smart' : 'selected'}_location_text`;
      if (this.currentInfo.state === 'connected') {
        stringKey = 'main_screen_current_location_text';
      }
      return this.localize(stringKey);
    },
    flagStyle() {
      let filename = (this.selectedLocation.country_code ? this.selectedLocation.country_code : 'XV') + '.svg';
      return chrome.extension.getURL('/images/flags/' + filename);
    },
    visibleLocations() {
      let locationList = [];
      if (this.currentInfo.smartLocation.id !== this.selectedLocation.id) {
        locationList.push(Object.assign({}, this.currentInfo.smartLocation, { type: 'smart' }));
      }

      for (let location of this.locationPicker.getRecentLocations(this.currentInfo.allLocationsList, this.currentInfo.recent_locations_ids)) {
        if (location.id !== this.selectedLocation.id && location.id !== this.currentInfo.smartLocation.id) {
          locationList.push(Object.assign({}, location, { type: 'recent' }));
        }
      }
      return locationList.slice(0, 2);
    },
    hintText() {
      let text = '';
      let connectingTimes = this.currentInfo.connectingTimes;
      let nConnections = this.currentInfo.connectingTimes.length;
      this.hintType = 'information';

      // Check for delayed connection
      if (
        nConnections === 5 && // Make sure there 4 connections in the history + the current connection
        this.currentInfo.state === 'connecting' && // It's still connecting
        this.utils.getTimeDelta(connectingTimes[nConnections - 1].startTime) > 10 && // it’s been more than 10 seconds
        this.utils.getTimeDelta(connectingTimes[nConnections - 1].startTime) > this.getAverageTopConnectionTime(2) * 1.25 &&
        this.discardHint === false// If the user already discarded the hint
      ) {
        text = 'hint_connection_delay_text';
        this.hintIcon = null;
      } else if (['ready', 'reconnecting', 'connecting'].includes(this.currentInfo.state) && this.currentInfo.hasInternet === false) { // GIVEN the user is NOT connected to the VPN and their internet connection is unavailable
        text = `hint_no_internet_${this.currentInfo.state}_text`;
        this.hintIcon = 'icon-13-block';
      } else if (this.currentInfo.state === 'reconnecting') {
        let networkLockStatus = this.currentInfo.preferences.traffic_guard_level ? 'on' : 'off';
        text = `hint_reconnecting_network_lock_${networkLockStatus}_text`;
        this.hintIcon = null;
      } else if (this.show4starHint === true) {
        text = 'rating_thanks4_text';
        this.hintType = 'green';
        this.hintIcon = 'icon-110-thanks';
      }
      return text;
    },
  },
  methods: {
    onDiscardRatingPrompt() {
      localStorage.setItem('lastDiscardDate', (new Date()).getTime());
      this.discardPrompt = true; // trigger computed
      this.forceRatingPrompt = false;
    },
    daysBetween(ts1, ts2) { // ToDo: Replace with moment.js
      return Math.round(Math.abs((+ts1) - (+ts2)) / 8.64e7);
    },
    getAverageTopConnectionTime: function (nLastConnections) {
      let sortedConnectingTimes = this.currentInfo.connectingTimes.sort((a, b) => b.delta - a.delta);
      let topNConnections = sortedConnectingTimes.slice(0, nLastConnections);

      return topNConnections.reduce((total, el) => total.delta + el.delta) / nLastConnections;
    },
    onSelectedLocationClick: function (ev) {
      if (!ev.target.classList.contains('current-location-more-icon')) {
        chrome.runtime.sendMessage({ connectToSelectedLocation: true });
      }
    },
    onPowerButtonClick: function () {
      switch (this.currentInfo.state) {
        case 'ready':
          chrome.runtime.sendMessage({ connectToSelectedLocation: true });
          break;
        case 'connecting':
        case 'reconnecting':
          if (parseInt(this.currentInfo.progress, 10) < 75) {
            chrome.runtime.sendMessage({ cancelConnection: true });
          }
          break;
        case 'connected':
          chrome.runtime.sendMessage({ disconnect: true });
          break;
        case 'speedtesting':
          chrome.runtime.sendMessage({ cancelSpeedTest: true });
          break;
        default:
          break;
      }
    },
    connectToLocation: function (location) {
      if (this.currentInfo.state === 'ready' || localStorage.getItem('hasCWCbefore')) {
        this.updateLocationAndConnect(location);
      } else {
        this.popupOptions = {
          id: 'home_change_location_popup',
          buttons: [
            {
              callback: () => { this.popupOptions = {}; },
            },
            {
              highlightIt: true,
              callback: () => {
                localStorage.setItem('hasCWCbefore', true);
                this.updateLocationAndConnect(location);
                this.popupOptions = {};
              },
            },
          ],
          isVisible: true,
          style: 'horizontal',
        };
      }
    },
    setPlaceholder() {
      let iconEl = document.querySelector('.current-location-flag');
      if (iconEl) {
        iconEl.src = chrome.extension.getURL('/images/flags/XV.svg');
      }
    },
  },
  components: {
    CircleProgressBar,
    Hint,
    messageItem,
    popup,
    mainHeader,
    ratingPrompt,
  },
  created() {
    let self = this;
    // ToDo: Improve this
    this.$parent.$parent.$on('discard-hint', function () {
      self.discardHint = true;
    });

    this.$root.$on('show-rating-prompt', () => {
      self.transitionDelay = '0s';
      self.forceRatingPrompt = true;
    });

    this.$on('show-4star-hint', () => {
      self.show4starHint = true;
      self.onDiscardRatingPrompt();
      setTimeout(() => {
        self.show4starHint = false;
      }, 4000);
    });
  },
};
</script>

<style lang="scss" scoped>
.no-highlight:hover, .no-highlight:active {
  color: inherit !important;
}

.content {

  .visual-state {
    g {
      fill: $gray-30;
      fill-opacity: 0.4;
      transition: fill-opacity 0.1s ease-in-out;
    }

    &-connected g {
      fill: $green-40;
      fill-opacity: 1;
      transition: fill-opacity 0.1s ease-in-out;
    }
  }

  .power-button {
    position: absolute;
    right: 0;
    bottom: 0;
    left: 0;
    margin: auto;
    height: 114px;
    width: 114px;

    &-container {
      position: relative;
      text-align: center;
      height: 54px;
    }

    .icon {
      position: absolute;
      top: 10px;
      right: 0;
      bottom: 0;
      left: 0;
      margin: auto;

      &-23-close {
        color: $gray-30;
      }

      &:hover {
        color: $gray-20;
      }
      &:active {
        color: $black-30;
      }
    }
  }

  .engine-state {
    text-align: center;
    font-size: 16px;
    margin-top: 16px;
    color: $gray-10;
  }

  .locations-container {
    margin-top: 50px;
    padding: 0 15px;

    .location-box {
      background: $gray-50;
      box-shadow: 0 3px 8px 1px rgba(0, 0, 0, 0.1);
      border-radius: 4px;

      &-text-type {
        font-size: 12px;
        color: $gray-20;
      }

      &-text-name {
        font-family: ProximaNova-Semibold;
        color: $black-20;
        font-size: 16px;
      }

      &:hover {
        background: $gray-40;

        .current-location-more-icon {
          color: $gray-20;
        }
      }

      &-connecting, &-reconnecting, &-disconnecting, &-speedtesting {
        box-shadow: none;

        &:hover {
          background: $gray-50;
        }
      }
    }

    .current-location {
      height: 50px;
      padding: 6px 15px;
      display: flex;
      align-items: center;
      justify-content: space-between;

      img {
        width: 29px;
        height: 29px;
      }

      .icon {
        font-size: 20px;

        &-disabled, &-disabled:hover {
          color: $gray-30;
        }
      }

      &-text {
        display: flex;
        flex-direction: column;
        height: 35px;
        margin-right: auto;
        margin-left: 10px;
        justify-content: space-evenly;
      }
    }

    .location-buttons {
      margin-top: 10px;
      display: flex;
      flex-direction: row;
      justify-content: space-between;
      
      .location-box {
        width: 155px;
        height: 80px;
        padding: 10px 15px;

        &-text-name { // do check for -webkit-line-clamp in the future
          margin-top: 5px;
          overflow: hidden;
          position: relative; 
          line-height: 20px;
          max-height: 40px; 
        }
      }
    }
  }

  .message-container {
    position: absolute;
    height: 85px;
    bottom: 0;
    width: 100%;
  }
}
</style>