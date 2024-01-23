<template>
  <div id="app">
    <div
      v-if="player.playing"
      class="now-playing"
      :class="getNowPlayingClass()"
    >
      <div class="now-playing__cover">
        <img
          :src="player.trackAlbum.image"
          :alt="player.trackTitle"
          class="now-playing__image"
          :class="{ fade: fade }" <!-- Conditionally apply fade class -->
        />
      </div>
      <div class="now-playing__details">
        <h1 class="now-playing__track" v-text="player.trackTitle"></h1>
        <h2 class="now-playing__artists" v-text="getTrackArtists"></h2>
      </div>
    </div>
    <div v-else class="now-playing" :class="getNowPlayingClass()">
    </div>
  </div>
</template>

<script>
import * as Vibrant from 'node-vibrant'
import props from '@/utils/props.js'

export default {
  name: 'NowPlaying',

  props: {
    auth: props.auth,
    endpoints: props.endpoints,
    player: props.player
  },

  data() {
    return {
      pollPlaying: '',
      playerResponse: {},
      playerData: this.getEmptyPlayer(),
      colourPalette: '',
      swatches: [],
      fade: false // Added fade variable
    }
  },

  computed: {
    getTrackArtists() {
      return this.player.trackArtists.join(', ')
    }
  },

  mounted() {
    this.setDataInterval()
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying)
  },

  methods: {
    async getNowPlaying() {
      let data = {};

      try {
        const response = await fetch(
          `${this.endpoints.base}/${this.endpoints.nowPlaying}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        );

        if (!response.ok) {
          throw new Error(`An error has occurred: ${response.status}`);
        }

        if (response.status === 204) {
          data = this.getEmptyPlayer();
          this.playerData = data;

          this.$nextTick(() => {
            this.$emit('spotifyTrackUpdated', data);
          });

          return;
        }

        data = await response.json();
        this.playerResponse = data;
      } catch (error) {
        this.handleExpiredToken();

        data = this.getEmptyPlayer();
        this.playerData = data;

        this.$nextTick(() => {
          this.$emit('spotifyTrackUpdated', data);
        });
      }
    },

    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle';
      const backgroundClass = this.player.playing ? '' : 'black-background';

      // Set fade to true when playing to trigger fade-in effect
      this.fade = this.player.playing;

      return `now-playing--${playerClass} ${backgroundClass}`;
    },

    getAlbumColours() {
      if (!this.player.trackAlbum?.image) {
        return;
      }

      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => {
          this.handleAlbumPalette(palette);
        });
    },

    getEmptyPlayer() {
      return {
        playing: false,
        trackAlbum: {},
        trackArtists: [],
        trackId: '',
        trackTitle: ''
      };
    },

    setDataInterval() {
      clearInterval(this.pollPlaying);
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying();
      }, 2500);
    },

    setAppColours() {
      document.documentElement.style.setProperty(
        '--color-text-primary',
        this.colourPalette.text
      );

      document.documentElement.style.setProperty(
        '--colour-background-now-playing',
        this.colourPalette.background
      );
    },

    handleNowPlaying() {
      if (
        this.playerResponse.error?.status === 401 ||
        this.playerResponse.error?.status === 400
      ) {
        this.handleExpiredToken();
        return;
      }

      if (this.playerResponse.is_playing === false) {
        this.playerData = this.getEmptyPlayer();
        return;
      }

      if (this.playerResponse.item?.id === this.playerData.trackId) {
        return;
      }

      // Introduce a delay before setting the fade class
      setTimeout(() => {
        this.fade = true;
      }, 500);

      this.playerData = {
        playing: this.playerResponse.is_playing,
        trackArtists: this.playerResponse.item.artists.map(
          artist => artist.name
        ),
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0].url
        }
      };
    },

    handleAlbumPalette(palette) {
      let albumColours = Object.keys(palette)
        .filter(item => {
          return item === null ? null : item;
        })
        .map(colour => {
          return {
            text: palette[colour].getTitleTextColor(),
            background: palette[colour].getHex()
          };
        });

      this.swatches = albumColours;

      this.colourPalette =
        albumColours[Math.floor(Math.random() * albumColours.length)];

      this.$nextTick(() => {
        this.setAppColours();
      });
    },

    handleExpiredToken() {
      clearInterval(this.pollPlaying);
      this.$emit('requestRefreshToken');
    }
  },

  watch: {
    auth: function(oldVal, newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying);
      }
    },

    playerResponse: function() {
      this.handleNowPlaying();
    },

    playerData: function() {
      this.$emit('spotifyTrackUpdated', this.playerData);

      this.$nextTick(() => {
        this.getAlbumColours();
      });

      // Set fade to false to trigger the fade effect
      this.fade = false;
    }
  }
};
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
