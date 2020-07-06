<template>
  <fragment>
    <v-tooltip bottom>
      <template v-slot:activator="{ on }">
        <v-icon
          v-if="auth"
          v-on="on"
          @click="openPlay()"
        >
          mdi-play-circle
        </v-icon>
      </template>
      <span>Play</span>
    </v-tooltip>
    <v-dialog
      v-model="dialog"
      :max-width="1024"
    >
      <v-card
        :elevation="0"
      >
        <v-toolbar
          dark
          color="primary"
        >
          <v-toolbar-title>Watch Session</v-toolbar-title>
          <v-spacer />
        </v-toolbar>
        <div ref="playterminal" />
        <v-container>
          <v-row no-gutters>
            <v-col
              cols="2"
              sm="6"
              md="2"
            >
              <v-card
                v-if="!disable"
                :elevation="0"
                class="pt-4"
                tile
              >
                <v-btn
                  large
                  color="primary"
                  @click="connect()"
                >
                  Watch
                </v-btn>
              </v-card>
              <v-card
                v-else
                :elevation="0"
                class="pt-4"
                tile
              >
                <v-card-text
                  v-if="sliderChange"
                >
                  {{ currentTime }}
                </v-card-text>

                <v-icon
                  v-if="!paused"
                  large
                  class="pl-12"
                  color="primary"
                  @click="paused = !paused"
                >
                  mdi-pause-circle
                </v-icon>
                <v-icon
                  v-else
                  large
                  class="pl-12"
                  color="primary"
                  @click="paused = !paused"
                >
                  mdi-play-circle
                </v-icon>
              </v-card>
            </v-col>
            <v-col
              cols="6"
              md="10"
            >
              <v-card
                :elevation="0"
                class="pt-4 pr-10"
                tile
              >
                <v-slider
                  v-model="currentTime"
                  class="ml-0"
                  min="0"
                  :max="totalLength"
                  :label="`${nowTimerDisplay} - ${endTimerDisplay}`"
                  @change="changeSliderTime"
                />
              </v-card>
            </v-col>
          </v-row>
        </v-container>
      </v-card>
    </v-dialog>
  </fragment>
</template>

<script>

import { Terminal } from 'xterm';
import { FitAddon } from 'xterm-addon-fit';
import moment from 'moment';
import 'moment-duration-format';
import 'xterm/css/xterm.css';

export default {
  name: 'SessionPlay',

  props: {
    uid: {
      type: String,
      required: true,
    },
    authenticated: {
      type: Boolean,
      required: true,
    },
  },

  data() {
    return {
      dialog: false,
      disable: false,
      currentTime: 0,
      totalLength: 0,
      endTimerDisplay: null,
      getTimerNow: null,
      paused: false,
      sliderChange: false,
      randomText: 'ndeio wendiowe nfie fwe',
      logs: [],
      frames: [],
      cols: 0,
      rows: 0,
    };
  },

  computed: {
    auth: {
      get() {
        return this.authenticated;
      },
    },

    length() {
      return this.logs.length;
    },

    nowTimerDisplay() {
      return this.getTimerNow;
    },
  },

  watch: {
    dialog(value) {
      if (!value) {
        this.close();
      }
    },
  },

  updated() {
    this.getTimerNow = this.getDisplaySliderInfo(this.currentTime).display;
  },

  async created() {
    if (this.auth) {
      await this.$store.dispatch('sessions/getLogSession', this.uid);
      this.logs = this.$store.getters['sessions/getLogSession'];
      this.totalLength = this.getDisplaySliderInfo(null).intervalLength;
      this.endTimerDisplay = this.getDisplaySliderInfo(null).display;
      this.getTimerNow = this.getDisplaySliderInfo(this.currentTime).display;
      this.cols = this.logs[0].width;
      this.rows = this.logs[0].height;
      this.frames = this.createFrames();
      // eslint-disable-next-line no-console
      console.log(this.frames[0].incTime);
    }
  },

  methods: {
    openPlay() {
      this.dialog = !this.dialog;
      this.xterm = new Terminal({ // instantiate
        cursorBlink: true,
        fontFamily: 'monospace',
        cols: this.cols,
        rows: this.rows,
      });
      this.fitAddon = new FitAddon(); // load fit
      this.xterm.loadAddon(this.fitAddon); // adjust screen in container
    },

    getDisplaySliderInfo(timeMs) {
      let interval;
      if (!timeMs) { // not params, will return metrics to max timelengtht
        const max = new Date(this.logs[this.length - 1].time);
        const min = new Date(this.logs[0].time);
        interval = max - min;
      } else { // it will format to the time argument passed
        interval = timeMs;
      }
      const duration = moment.duration(interval, 'milliseconds').format('hh:mm:ss:SS', {
        trim: false,
      });
      return {
        display: duration, // format to slider label
        intervalLength: interval, // length of slider in Ms
      };
    },

    createFrames() { // create cumulative frames for the exibition in slider
      let message = '';
      let time = 0;
      const arrFrames = [{
        incMessage: message,
        incTime: time,
      }];
      for (let i = 0; i < this.logs.length - 1; i += 1) {
        const future = new Date(this.logs[i + 1].time);
        const now = new Date(this.logs[i].time);
        const interval = future - now;
        message += this.logs[i].message;
        time += interval;
        arrFrames.push({
          incMessage: message,
          incTime: moment.duration(time, 'milliseconds').asMilliseconds(),
        });
      }
      return arrFrames;
    },

    timer() { // Increments the slider
      if (!this.paused) {
        if (this.currentTime >= this.totalLength) return;
        this.currentTime += 100;
      }
      this.iterativeTimer = setTimeout(this.timer.bind(null), 100);
    },

    connect() {
      this.disable = true;
      this.xterm.open(this.$refs.playterminal);
      this.$nextTick(() => this.fitAddon.fit());
      this.fitAddon.fit();
      this.xterm.focus();
      // this.print(0, this.logs);
      this.printTest();
      this.timer();
      if (this.xterm.element) { // check already existence
        this.xterm.reset();
      }
    },

    changeSliderTime() { // Moving the Slider
      this.sliderChange = true;
    },

    close() {
      if (this.xterm) this.xterm.dispose();
      this.disable = false;
      clearInterval(this.iterativePrinting);
      clearInterval(this.iterativeTimer);
      this.currentTime = 0;
      this.paused = false;
    },

    printTest() { // delete
      this.xterm.write(this.frames[this.frames.length - 1].incMessage);
      setTimeout(() => { this.xterm.write('\u001Bc'); this.xterm.write('Another Random Message\n'); }, 4000);
    },

    print(i, logsArray) {
      if (!this.paused) {
        this.sliderChange = false;
        this.xterm.write(`${logsArray[i].message}`);
        if (i === logsArray.length - 1) return;
        const nowTimerDisplay = new Date(logsArray[i].time);
        const future = new Date(logsArray[i + 1].time);
        const interval = future - nowTimerDisplay;
        this.iterativePrinting = setTimeout(this.print.bind(null, i + 1, logsArray), interval);
      } else { // try to execute back every 100 ms
        this.iterativePrinting = setTimeout(this.print.bind(null, i, logsArray), 100);
      }
    },
  },
};

</script>
