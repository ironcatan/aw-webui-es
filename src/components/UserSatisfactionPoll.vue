<template lang="pug">
  div
    b-alert(v-if="isPollVisible", variant="info", show)
      button(type="button", class="close", @click="isPollVisible=false") &times;
      form
        p
          | {{ $t('poll.question') }}
        div(class="radio-options")
          div(v-for="i in options", class="option-group")
            input(type="radio", :id="'option' + i", name="rating", :value="i", v-model="rating")
            br
            label(:for="'option' + i", style="display: block")
              | {{ i }}
      div(style="display: flex; justify-content: space-between")
        a(@click="dontShowAgain" href="#")
          | {{ $t('poll.dontShowAgain') }}
        input(type="submit" :value="$t('poll.submit')" @click="submit")

    b-alert(v-if="isPosFollowUpVisible", variant="info" show)
      button(type="button", class="close", @click="isPosFollowUpVisible=false") &times;
      p
        | {{ $t('poll.posTitle') }}
        br
        | {{ $t('poll.posSubtitle') }}
      ul.small
        li
          | {{ $t('poll.supportUsOn') }} #[a(href="https://www.patreon.com/erikbjare") {{ $t('poll.patreon') }}] {{ $t('poll.or') }} #[a(href="https://opencollective.com/activitywatch") {{ $t('poll.openCollective') }}] ({{ $t('poll.orByOther') }} #[a(href="https://activitywatch.net/donate/") {{ $t('poll.otherDonationMethods') }}]).
        li
          | {{ $t('poll.tellFriends') }}
        li
          | {{ $t('poll.postSocialPrefix') }} #[a(href="https://twitter.com/ActivityWatchIt") {{ $t('poll.twitter') }}] {{ $t('poll.and') }} #[a(href="https://www.facebook.com/ActivityWatch") {{ $t('poll.facebook') }}].
        //li
          | Fill out the #[a(href="https://forms.gle/q2N9K5RoERBV8kqPA") feedback form].
        li
          | {{ $t('poll.rateUsOn') }} #[a(href="https://alternativeto.net/software/activitywatch/about/") {{ $t('poll.alternativeTo') }}] {{ $t('poll.and') }} #[a(href="https://play.google.com/store/apps/details?id=net.activitywatch.android") {{ $t('poll.googlePlay') }}].
        li
          | {{ $t('poll.joinOurPrefix') }} #[a(href="https://discord.gg/vDskV9q") {{ $t('poll.discordServer') }}].
        li
          | {{ $t('poll.signUpPrefix') }} #[a(href="http://eepurl.com/cTU6QX") {{ $t('poll.newsletter') }}] {{ $t('poll.newsletterSuffix') }}

    b-alert(v-if="isNegFollowUpVisible", variant="info" show)
      button(type="button", class="close", @click="isNegFollowUpVisible=false") &times;
      | {{ $t('poll.negTitle') }}
      ul
        li
          | {{ $t('poll.fillFormPrefix') }} #[a(href="https://forms.gle/q2N9K5RoERBV8kqPA") {{ $t('poll.feedbackForm') }}].
        li
          | {{ $t('poll.voteFeaturesPrefix') }} #[a(href="https://forum.activitywatch.net/c/features") {{ $t('poll.forumLink') }}].
</template>

<style scoped>
.radio-options {
  display: flex;
  justify-content: space-around;
}

.option-group {
  text-align: center;
}

ul {
  margin: 0;
}
</style>

<script lang="ts">
import { range } from 'lodash/fp';
import moment from 'moment';

import { useSettingsStore } from '~/stores/settings';

const NUM_OPTIONS = 10;
// BACKOFF_PERIOD is how many seconds to wait to show the poll again if the user closed it
const BACKOFF_PERIOD = 7 * 24 * 60 * 60;
// The following may be used for testing
// const INITIAL_WAIT_PERIOD = 1;
// const BACKOFF_PERIOD = 1;

export default {
  name: 'user-satisfaction-poll',
  data() {
    return {
      isPollVisible: false,
      isPosFollowUpVisible: false,
      isNegFollowUpVisible: false,
      // options is an array of [1, ..., NUM_OPTIONS]
      options: range(1, NUM_OPTIONS + 1),
      rating: null,
    };
  },
  computed: {
    data: {
      get() {
        const settingsStore = useSettingsStore();
        return settingsStore.userSatisfactionPollData;
      },
      set(value) {
        const settingsStore = useSettingsStore();
        const data = settingsStore.userSatisfactionPollData;
        settingsStore.update({
          userSatisfactionPollData: { ...data, ...value },
        });
      },
    },
  },
  async mounted() {
    if (!this.data.isEnabled) {
      return;
    }

    // Show poll if enough time has passed
    if (moment() >= moment(this.data.nextPollTime)) {
      this.data.timesPollIsShown = this.data.timesPollIsShown + 1;
      this.isPollVisible = true;
      this.data.nextPollTime = moment().add(BACKOFF_PERIOD, 'seconds');
    }

    // Show the poll a maximum of 3 times
    if (this.data.timesPollIsShown > 2) {
      this.data.isEnabled = false;
    }
  },
  methods: {
    submit() {
      this.isPollVisible = false;
      this.data = { ...this.data, isEnabled: false };

      if (parseInt(this.rating) >= 6) {
        this.isPosFollowUpVisible = true;
      } else {
        this.isNegFollowUpVisible = true;
      }
    },
    dontShowAgain() {
      this.isPollVisible = false;
      this.data = { ...this.data, isEnabled: false };
    },
  },
};
</script>
