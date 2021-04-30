<template>
  <v-layout
    column
    justify-center
    align-center
  >
    <v-flex
      xs12
      sm8
      md6
    >
      <v-card>
        <v-card-title class="headline">
          Welcome to the extremely welll coded album rainbow maker
        </v-card-title>
        <v-card-text>
          <p>
            Find a bug? WELL ITS ACTUALLY A FEATURE SIKEEEE!!! You can see the code <a
              href="https://github.com/reteps/spotify-cards"
              target="_blank"
              title="mfs always hovering over my text smh"
            >
              here
            </a>.
          </p>
          <div class="text-xs-right">
            <em><small>&mdash; Peter Stenger</small></em>
          </div>
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn
            color="primary"
            :href="authURL"
          >
            Login with spotify <v-icon class="ml-2">fab fa-spotify</v-icon>
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-flex>
  </v-layout>
</template>

<script>
export default {
  data() {
    return {
      callback: 'https://spotify-poster.herokuapp.com/callback',
      clientID: 'e8f9e43911414da7ba357c2119d76214'
    }
  },
  computed: {
    authURL() {
      return `https://accounts.spotify.com/authorize?client_id=${this.clientID}&response_type=token&redirect_uri=${this.callback}&scope=user-library-read`
    }
  }
}
</script>
