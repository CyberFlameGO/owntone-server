<template>
  <section>
    <div v-if="now_playing.id > 0" class="fd-is-fullheight">
      <div class="fd-is-expanded">
        <cover-artwork
          :artwork_url="now_playing.artwork_url"
          :artist="now_playing.artist"
          :album="now_playing.album"
          class="fd-cover-image fd-has-action"
          @click="open_dialog(now_playing)"
        />
      </div>
      <div class="fd-has-padding-left-right">
        <div class="container has-text-centered">
          <p class="control has-text-centered fd-progress-now-playing">
            <Slider
              ref="slider"
              v-model="item_progress_ms"
              :min="0"
              :max="state.item_length_ms"
              :step="1000"
              :tooltips="false"
              :disabled="state.state === 'stop'"
              :classes="{ target: 'seek-slider' }"
              @change="seek"
              @start="start_dragging"
              @end="end_dragging"
            />
            <!--range-slider
              class="seek-slider fd-has-action"
              min="0"
              :max="state.item_length_ms"
              :value="item_progress_ms"
              :disabled="state.state === 'stop'"
              step="1000"
              @change="seek" >
            </range-slider-->
          </p>
          <p class="content">
            <span
              >{{ $filters.durationInHours(item_progress_ms) }} /
              {{ $filters.durationInHours(now_playing.length_ms) }}</span
            >
          </p>
        </div>
      </div>
      <div class="fd-has-padding-left-right">
        <div class="container has-text-centered fd-has-margin-top">
          <h1 class="title is-5">
            {{ now_playing.title }}
          </h1>
          <h2 class="title is-6">
            {{ now_playing.artist }}
          </h2>
          <h2
            v-if="composer"
            class="subtitle is-6 has-text-grey has-text-weight-bold"
          >
            {{ composer }}
          </h2>
          <h3 class="subtitle is-6">
            {{ now_playing.album }}
          </h3>
        </div>
      </div>
    </div>
    <div v-else class="fd-is-fullheight">
      <div
        class="fd-is-expanded fd-has-padding-left-right"
        style="flex-direction: column"
      >
        <div class="content has-text-centered">
          <h1 class="title is-5">Your play queue is empty</h1>
          <p>Add some tracks by browsing your library</p>
        </div>
      </div>
    </div>
    <modal-dialog-queue-item
      :show="show_details_modal"
      :item="selected_item"
      @close="show_details_modal = false"
    />
  </section>
</template>

<script>
import ModalDialogQueueItem from '@/components/ModalDialogQueueItem.vue'
//import RangeSlider from 'vue-range-slider'
import Slider from '@vueform/slider'
import CoverArtwork from '@/components/CoverArtwork.vue'
import webapi from '@/webapi'
import * as types from '@/store/mutation_types'

export default {
  name: 'PageNowPlaying',
  components: {
    ModalDialogQueueItem,
    //    RangeSlider,
    Slider,
    CoverArtwork
  },

  data() {
    return {
      item_progress_ms: 0,
      interval_id: 0,
      is_dragged: false,

      show_details_modal: false,
      selected_item: {}
    }
  },

  computed: {
    state() {
      return this.$store.state.player
    },

    now_playing() {
      return this.$store.getters.now_playing
    },

    settings_option_show_composer_now_playing() {
      return this.$store.getters.settings_option_show_composer_now_playing
    },

    settings_option_show_composer_for_genre() {
      return this.$store.getters.settings_option_show_composer_for_genre
    },

    composer() {
      if (this.settings_option_show_composer_now_playing) {
        if (
          !this.settings_option_show_composer_for_genre ||
          (this.now_playing.genre &&
            this.settings_option_show_composer_for_genre
              .toLowerCase()
              .split(',')
              .findIndex(
                (elem) =>
                  this.now_playing.genre.toLowerCase().indexOf(elem.trim()) >= 0
              ) >= 0)
        ) {
          return this.now_playing.composer
        }
      }
      return null
    }
  },

  watch: {
    state() {
      if (this.interval_id > 0) {
        window.clearTimeout(this.interval_id)
        this.interval_id = 0
      }
      this.item_progress_ms = this.state.item_progress_ms
      if (this.state.state === 'play') {
        this.interval_id = window.setInterval(this.tick, 1000)
      }
    }
  },

  mounted: function () {
    console.log(this.$refs.slider)
  },

  created() {
    this.item_progress_ms = this.state.item_progress_ms
    webapi.player_status().then(({ data }) => {
      this.$store.commit(types.UPDATE_PLAYER_STATUS, data)
      if (this.state.state === 'play') {
        this.interval_id = window.setInterval(this.tick, 1000)
      }
    })
  },

  unmounted() {
    if (this.interval_id > 0) {
      window.clearTimeout(this.interval_id)
      this.interval_id = 0
    }
  },

  methods: {
    tick: function () {
      if (!this.is_dragged) {
        this.item_progress_ms += 1000
      }
    },

    start_dragging: function () {
      console.log('@start')
      this.is_dragged = true
    },

    end_dragging: function () {
      console.log('@end')
      this.is_dragged = false
    },

    seek: function (newPosition) {
      webapi.player_seek_to_pos(newPosition).catch(() => {
        this.item_progress_ms = this.state.item_progress_ms
      })
    },

    open_dialog: function (item) {
      this.selected_item = item
      this.show_details_modal = true
    }
  }
}
</script>

<style></style>
