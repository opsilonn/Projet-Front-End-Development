<template>
    <div class="indigo darken-3 height100">
      <v-card
        class="height100"
        dark
      >
        <!-- Progress bar -->
        <v-progress-linear
          v-model="timerToProgress"
          :color="timerToGradient"
          :background-color="timerToGradientDarken"/>
        <label
          class="d-flex justify-end align-end pr-3"
          :style="'color: ' + timerToGradient"
        >
          <h2>{{timerRemainingSec.toFixed(1)}}</h2>
          <h3>s</h3>
        </label>

        <!-- Questions -->
        <h3 v-html="trivia.question" class="pa-0 pl-5 pr-5" align="center"/>
        <v-row>
          <v-col
            cols="6"
            v-for="(answer, index) in trivia.answers" v-bind:key="index"
            :class="'pt-6 ' + (index%2 ? 'pr-12 pl-4' : 'pl-12 pr-4')"
          >
            <v-hover v-slot:default="{ hover }">
              <!-- Choosing the answer -->
              <v-card v-if="(step&1) === 1"
                :style="'background-color: ' + (hover ? themes.DarkLighter : themes.DarkLight)"
                :elevation="hover ? 12 : 0"
                @click="chooseAnswer(answer)"
                >
                <v-card-text v-html="answer.answer"/>
              </v-card>

              <!-- Waiting for result -->
              <v-card v-else-if="(step&2) !== 2"
                :style="'background-color: ' + (yourAnswer === answer ? themes.DarkLighter : themes.DarkLight)"
                :elevation="yourAnswer === answer ? 12 : 0"
                >
                <v-card-text v-html="answer.answer"/>
              </v-card>

              <!-- Displaying the result -->
              <v-card v-else
                :style="'background-color: ' + (answer.value ? themes.Success : themes.Failure)"
                :elevation="yourAnswer === answer ? 12 : 0"
                >
                <v-card-text v-html="answer.answer"/>
              </v-card>
            </v-hover>
          </v-col>
        </v-row>
      </v-card>
    </div>
</template>

<script>
import EventBus from '@/EventBus.js'

import { mapState, mapGetters, mapMutations } from 'vuex'

export default {
  name: 'GameQuestions',
  data: () => ({
    timer: {
      length: 0,
      begin: 0,
      end: 0,
      remaining: 0
    },
    trivia: {},
    step: 1,
    yourAnswer: { answer: '' },
    showResults: false
  }),
  computed: {
    // States
    ...mapState('themes', ['themes']),
    ...mapState('parties', ['parties']),
    ...mapState('dungeons', ['dungeons']),
    ...mapState('rounds', ['rounds']),

    // Getters
    ...mapGetters('parties', ['getPartyById']),
    ...mapGetters('dungeons', ['getDungeonsByPartyId', 'getLastDungeonByPartyId']),
    ...mapGetters('rounds', ['getLastRoundByDungeonId']),

    // Custom
    partyId () {
      return parseInt(this.$route.params.partyId)
    },
    dungeon () {
      return this.getLastDungeonByPartyId(this.partyId) || { category: '0', difficulty: 'none', roundTime: 0, number: '0' }
    },
    round () {
      return this.getLastRoundByDungeonId(this.dungeon.id) || { number: '0' }
    },
    timerRemainingSec () {
      return Math.abs((this.timer.remaining || 0.0) / 1000)
    },
    timerToProgress () {
      return ((this.timer.end - this.timer.begin) - this.timer.remaining) / (this.timer.end - this.timer.begin) * 100
    },
    timerToRed () {
      return parseFloat((this.timerToProgress / 100 * 255).toFixed(0))
    },
    timerToGreen () {
      return parseFloat(255 - this.timerToRed)
    },
    timerToGradient () {
      return '#' + this.timerToRed.toString(16) + this.timerToGreen.toString(16) + '00'
    },
    timerToGradientDarken () {
      return '#' + (this.timerToRed / 255 * 153).toString(16) + (this.timerToGreen / 255 * 153).toString(16) + '00'
    }
  },
  methods: {
    // Mutations
    ...mapMutations('rounds', ['setRoundAnswer']),

    // Method called when the user chooses an answer
    chooseAnswer (answer) {
      this.yourAnswer = answer

      // We stop the Timer
      EventBus.$emit('timer-stop', { timer: this.timer })
    },

    // Event handlers
    onTimer_start ({ timer, trivia }) {
      this.showResults = false
      this.yourAnswer = {}
      this.timer = timer
      this.trivia = trivia
      this.step = 1
    },
    onTimer_update ({ timer }) {
      this.timer = timer
    },
    onTimer_end ({ timer }) {
      this.timer = timer
      this.step = 0
    },
    onTimer_reset ({ timer }) {
      this.timer = timer
      this.step = 2

      this.setRoundAnswer({ roundId: this.round.id, answer: this.yourAnswer })
    }
  },
  created () {
    EventBus.$on('timer-start', ({ timer, trivia }) => this.onTimer_start({ timer, trivia }))
    EventBus.$on('timer-update', ({ timer }) => this.onTimer_update({ timer }))
    EventBus.$on('timer-end', ({ timer }) => this.onTimer_end({ timer }))
    EventBus.$on('timer-reset', ({ timer }) => this.onTimer_reset({ timer }))
  }
}
</script>
