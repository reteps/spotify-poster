<template>
  <v-layout>
    <v-flex class="text-center">
      <img
        src="/v.png"
        alt="Vuetify.js"
        class="mb-5"
      >
      <div v-if="!songsLoaded">
        <v-card>
          <v-card-title></v-card-title>
          <v-card-text>
            <p class="text-center headline">Select a playlist</p>
            <p>Playlist not there? Make sure it is public and not collaborative.</p>

        <v-list>
        <v-list-item-group v-model="selectedPlaylist">
          <v-list-item v-for="playlist in playlists" :key="playlist.id">
            <v-list-item-avatar tile><v-img :src="playlist.images[0].url"></v-img></v-list-item-avatar>
            <v-list-item-content>
              <v-list-item-title v-text="playlist.name"></v-list-item-title>
            </v-list-item-content>
          </v-list-item>
        </v-list-item-group>
        </v-list>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn @click='loadTracks' :disabled="selectedPlaylist == null">Next Step</v-btn>
          <v-progress-circular v-if="selectedPlaylist != null && !songsLoaded && loadingProgress !== null" :value="(loadingProgress / maxRequests || 0) * 100"></v-progress-circular>
        </v-card-actions>
        </v-card>
      </div>
      <div v-else>
        <v-list-item @click='selectedPlaylist = null;loadingProgress = 0' :key="playlist.id">
          <v-list-item-avatar tile><v-img :src="playlist.images[0].url"></v-img></v-list-item-avatar>
          <v-list-item-content>
            <v-list-item-title v-text="playlist.name"></v-list-item-title>
          </v-list-item-content>
        </v-list-item>
        <v-form ref="form">
          <v-row>
          <v-col>
          <v-select v-model="sortType" label="Sort By ..." :items="Object.keys(sortFunctions)" required></v-select>
          </v-col>
                    <v-col>
            <v-select v-model="arrangeType" label="Arrange By ..." :items="Object.keys(arrangeFunctions)" required></v-select>
          </v-col>
          </v-row>
          <v-row>
            <v-col>
              <v-checkbox v-model="albumDupCheck" label="Remove duplicate album covers"></v-checkbox>
              <v-checkbox class="ml-5 mt-0" v-if="albumDupCheck" v-model="aggressiveDupCheck" label="Aggressive (based on color)"></v-checkbox>
            </v-col>
            <v-col>
              <v-checkbox v-model="codeCheck" label="Display Spotify Codes"></v-checkbox>
              <v-checkbox class="ml-5 mt-0" v-if="codeCheck" v-model="albumCodeCheck" label="Use Album Code instead of Track Code"></v-checkbox>
              <v-checkbox class="mt-0" v-model="coverCheck" label="Display Cover images" checked></v-checkbox>
            </v-col>
          </v-row>
          <v-row>
          <v-col>
          <v-radio-group v-model="selectedFormat">
            <v-radio label='Aspect Ratio' value='Aspect Ratio' key='Aspect Ratio'></v-radio>
            <v-radio label='Columns' value='Columns' key='Columns'></v-radio>
          </v-radio-group>
          </v-col>
          <v-col>
          <div v-if='selectedFormat == "Aspect Ratio"'>
            <v-radio-group v-model="selectedAspect">
              <v-radio v-for="(aspect,i) in aspects" :label="aspect.text" :value="i" :key="aspect.text"></v-radio>
            </v-radio-group>
          </div>
          <div v-if='selectedFormat == "Columns"'>
            <v-text-field type="number" label="# of Columns" v-model="columns"></v-text-field>
          </div>
          </v-col>
          </v-row>
          <v-row>
            <v-col>
              <v-btn @click='processTracks'>{{ `Generate using ${raw.length} tracks` }} </v-btn><v-progress-circular v-if="processing" :value=" Math.round(generateProgress / 5) * 5"></v-progress-circular>
            </v-col>
            <v-col v-if="!processing && unprocessed.length == raw.length">
              <v-text-field type="number" label="Margin (px)" v-model="margin"></v-text-field>
            </v-col>
            <v-col v-if="!processing && unprocessed.length == raw.length">
              <v-card flat color="transparent" class="headline">Image Quality
              <v-card-text>
                <v-slider v-model="quality" max="100" min="10"></v-slider>
                <div> With the current settings, you will generate a {{calcWidth}}px x {{ calcHeight }}px image (~{{ Math.round(calcWidth * calcHeight * (32/8) * (1/3) / (1024*1024)) }}mb)</div>
                <v-row>
                <v-btn @click='exportAsImage'>Export to PNG</v-btn>
                <v-tooltip bottom>
                  <template v-slot:activator="{ on, attrs}">
                  <v-icon class="ml-2" v-bind="attrs" v-on="on">{{ status == '' ? 'fa fa-exclamation-triangle' : 'fa fa-tasks' }} </v-icon>
                  </template>
                  <span> {{  status == '' ? 'Make sure you have scrolled down the page all the way to render all the elements!' : 'This could take a while, be patient!' }} </span>
                </v-tooltip>
                  <span class="body-2 ml-2"> {{ status }} </span>
                </v-row>
                <v-row>
                  <v-col>
                  <v-text-field type="number" label="rows per page (please wait, very laggy)" v-model="rawRowsPer"></v-text-field>
                  </v-col>
                  <v-col>
                  <v-text-field type="number" label="cols per page (please wait, very laggy)" v-model="rawColsPer"></v-text-field>
                  </v-col>
                  <v-col>
                  <v-btn @click='exportToPDF'>Export in sections (WARNING VERY LAGGY)</v-btn>
                  <v-btn @click='exportAsCanvas'>Export to canvas</v-btn>
                  </v-col>
                </v-row>
              </v-card-text>
              </v-card>
            </v-col>
          </v-row>
        </v-form>
        <div id="mega-image" v-if="!processing">
          <div class="d-flex flex-row" v-for="(_, rowChunk) in rowSections">
            <div class="export-section d-flex flex-column" v-for="(_, colChunk) in colSections" :style="`max-width: ${100/(colCount / colsPer)}%; width: ${100/(colCount / colsPer)}%;`">
              <div class="d-flex flex-row" v-for="(_, row) in rowsPer">
                <ImgGroup v-for="(_, col) in colsPer" v-if="true && (rowChunk * rowsPer) + row <= rowCount && (colChunk * colsPer) + col < colCount && (rowChunk * colCount * rowsPer) + (colsPer * colChunk) + (row * colCount) + col <= processed.length"
                  @hide='removeTrack' :showCover="coverCheck" :showCode="codeCheck" :margin="parseInt(margin)" :cols="colsPer" :song="processed[(rowChunk * colCount * rowsPer) + (colsPer * colChunk) + (row * colCount) + col]"></ImgGroup>
              </div>
            </div>
          </div>
        </div>
      </div>
    </v-flex>
  </v-layout>
</template>
<script>

const ColorThief = require('colorthief').default;
var tinycolor = require("tinycolor2");
import domtoimage from '../assets/dom-to-image-fix';
const _ = require('lodash')
import ImgGroup from '~/components/ImgGroup.vue'
export default {
  name: 'hi',
  components: {
    ImgGroup
  },
  data() {
    return {
      accessToken: null,
      playlists: [],
      selectedPlaylist: null,
      maxRequests: 0,
      loadingProgress: null,

      processing: false,

      callback: 'https://spotify-poster.herokuapp.com/callback',
      clientID: 'e8f9e43911414da7ba357c2119d76214',

      raw: [], // Initial Songs Data
      unprocessed: [], // Song data with album info
      processed: [], //  Song data filtered and sorted
      removed: [],
      // form data
      aggressiveDupCheck: 1,
      albumDupCheck: 1,
      coverCheck: 1,
      codeCheck: 1,
      albumCodeCheck: 0,
      sortType: 'color',
      arrangeType: 'diagonal',
      oldSort: '',
      generateProgress: 0,
      sortFunctions: {
        'color': (a, b) => {
          if (!a) { return -1 }
          if (!b) { return 1 }
          let aC = a.color.toHsl()
          let bC = b.color.toHsl()
          return aC.h == bC.h ? (aC.s > bC.s ? 1 : -1) : (aC.h > bC.h ? 1 : -1)
        },
      },
      arrangeFunctions: {
        'diagonal': this.getDiagonal,
        'vertical': this.getVertical
      },
      formats: ['Aspect Ratio', 'Columns'],
      selectedFormat: 'Aspect Ratio',
      aspects: [{text: 'Grid (1:1)', width: 1, height: 1}, {text: 'Spotify Code Grid (1:4)', width: 1, height: 4}, {text: 'Square With Codes (5:4)', width: 5, height: 4}],
      selectedAspect: 0,
      columns: null,
      margin: 0,
      rowCount: 0,
      colCount: 0,
      quality: 50,
      status: '',
      rawRowsPer: 1,
      rawColsPer: 1,
    }
  },
  mounted() {
    this.accessToken = this.getHashVal('access_token')
    this.accessToken ? this.getPlaylists() : console.log('error no access token')
  },
  computed: {
    rowsPer() {
      return parseInt(this.rawRowsPer) || this.rowCount
    },
    colsPer() {
      return parseInt(this.rawColsPer) || this.colCount
    },
    rowSections() {
      return Math.ceil(this.rowCount / this.rowsPer)
    },
    colSections() {
      return Math.ceil(this.colCount / this.colsPer)
    },
    songsLoaded() {
      return this.selectedPlaylist != null && this.loadingProgress == this.maxRequests
    },
    authURL() {
      return `https://accounts.spotify.com/authorize?client_id=${this.clientID}&response_type=token&redirect_uri=${this.callback}`
    },
    playlist() {
      return this.playlists[this.selectedPlaylist]
    },
    calcWidth() {
      return (this.margin*this.colCount*2 + 640*this.colCount)*this.quality/100
    },
    calcHeight() {
      return (this.margin*this.rowCount*2 + (this.coverCheck ? 640*this.rowCount : 0) + (this.codeCheck ? 160*this.rowCount : 0)) * this.quality/100
    },
    calcSectionWidth() {
      return this.margin * this.colsPer * 2 + 640 * this.colsPer * this.quality / 100
    },
    calcSectionHeight() {
      return (this.margin*this.rowsPer*2 + (this.coverCheck ? 640*this.rowsPer : 0) + (this.codeCheck ? 160*this.rowsPer : 0)) * this.quality/100
    }
  },
  watch: {
    albumCodeCheck: function (isChecked) {
      this.unprocessed = this.unprocessed.map(e => {
        let rgbColor = e.color
        let codeColor = rgbColor.isLight() ? 'black' : 'white'
        let backgroundColor = rgbColor.toHex()
        e.code = `https://scannables.scdn.co/uri/plain/png/${backgroundColor}/${codeColor}/640/${isChecked ? e.albumURI : e.trackURI}`
        return e
      })
    },
  },
  methods: {
    getHashVal(key) {
      let regex = new RegExp(`${key}=([^&]*)`)
      let matches = location.hash.match(regex)
      return matches ? matches[1] : null;
    },
    getPlaylists() {
      console.log('Retrieving Playlists')
      let vm = this;
      this.$axios.get('https://api.spotify.com/v1/me/playlists', { headers: {Authorization: `Bearer ${this.accessToken}`}})
      .then(res => res.data.items)
      .then(playlists => {
        vm.playlists = playlists.filter(p => p.images.length > 0)
      }).catch(err => {
        if (err.response.status == 401) {
          window.location = vm.authURL
        }
      })
    },
    exportToPDF() {
      let scale = this.calcSectionWidth / document.getElementsByClassName('export-section')[0].offsetWidth
      let settings = {
        width: this.calcSectionWidth,
        height: this.calcSectionHeight,
        style: {
          transform: 'scale('+scale+')',
          transformOrigin: 'top left',
          width: this.calcSectionWidth + 'px',
          height: this.calcSectionHeight + 'px'
        }
      }
      let parts = [...document.querySelectorAll('.export-section')]
      parts.forEach((section,i) => {
        domtoimage.toBlob(section, settings).then((blob) => {
        let streamSaver = null;
        if (process.client) {
          streamSaver = require('streamsaver')
        }
        const stream = streamSaver.createWriteStream(`poster-section-${i+1}.png`, {
          size: blob.size
        })
        blob.stream().pipeTo(stream)
      })
      })
    },

    exportAsImage() {
      this.status = `downloading ${this.processed.length} images ...`
      console.log('New height is', this.calcHeight)
      console.log('New width is', this.calcWidth)
      console.log('Old width is', document.getElementById('mega-image').offsetWidth)
      let scale = this.calcWidth / document.getElementById('mega-image').offsetWidth
      console.log('Scaling up by a factor of', scale)
      let settings = {width: this.calcWidth, height: this.calcHeight,
      style: {
        transform: 'scale('+scale+')',
        transformOrigin: 'top left',
        width: this.calcWidth + 'px',
        height: this.calcHeight + 'px'
      }
    }
    let vm = this;
      domtoimage.toBlob(document.getElementById('mega-image'), settings)
    .then(function (blob) {
        vm.status = `preparing to save ...`
        let streamSaver = null;
        if (process.client) {
          streamSaver = require('streamsaver')
        }
        const stream = streamSaver.createWriteStream(`poster-${vm.calcWidth}-${vm.calcHeight}.png`, {
          size: blob.size
        })
        vm.status = `downloading ...`
        blob.stream().pipeTo(stream)
        .then(() => vm.status = `Success!`)

    });
    },
    loadTracks() {
      this.loadingProgress = 0
      this.maxRequests = Math.ceil(this.playlist.tracks.total / 100)
      this.raw = []
      this.unprocessed = []
      this.processed = []

      console.log('Loading Tracks....')
      let selectedPlaylistTrackURL = this.playlist.tracks.href
      let offsets = [...Array(this.maxRequests).keys()]
      let vm = this;
      offsets.forEach(n => {
        this.$axios.get(selectedPlaylistTrackURL, { params: {offset: n*100}, headers: {Authorization: `Bearer ${this.accessToken}`}}).then(res => {
          vm.loadingProgress++
          vm.raw = vm.raw.concat(...res.data.items)
        }).catch(err => {
          if (err.response.status == 401) {
            window.location = vm.authURL
          }
        })
      })
    },
    getDiagonal(songList, width, height) {
      songList = songList.sort(this.sortFunctions[this.sortType])
      if ((height - 1) * width >= songList.filter(s => s != null).length) {
        height -= 1
        this.rowCount -= 1
      }
      console.log('Rearranging in diagonal order...')
      // let empty = {cover: null, code: null, color: tinycolor({ h: 1, s: 0, l: .5 }), id: '', albumID: '', artistNames: '', albumName: '' }
      let endingCoords = []
      for( let k = 0 ; k <= width + height - 2; k++ ) {
        for( let j = 0 ; j <= k ; j++ ) {
            let i = k - j
            if( i < height && j < width ) {
              endingCoords.push(i * width + j)
            }
        }
      }
      let noNullSongList = songList.filter(s => s != null)
      let result = Array(height*width)
      let offset = 0;
      for (let i = 0; i < height*width; i++) {
        // calculate back to 1d
        // if element was removed,
        let pos = endingCoords[i]
        let song = i >= noNullSongList.length ? null : noNullSongList[i]
        result[pos] = song
      }
      console.log('Done rearranging.')
      return result
    },
    getVertical(songList, width, height) {
      return songList
    },
    generateProcessedImages(initialSongs, unprocessed) {
      this.generateProgress = 0
      let vm = this;
      return new Promise((resolve, reject) => {
        if (initialSongs.length == unprocessed.length) { // basically check if we need to generate all this stuff or it was already done
          resolve(unprocessed)
          return;
        }
        console.log('Calculating Primary Colors')
        const colorThief = new ColorThief();
        let tempSongs = []
        initialSongs.forEach((s,i) => {
          const img = new Image();
          img.crossOrigin = 'Anonymous'

          img.src = s.track.album.images[2].url;

          img.addEventListener('load', function() {
            vm.generateProgress += 100 / initialSongs.length
            let possibleColors = colorThief.getPalette(img, 4, 1).map(c => tinycolor(`rgb(${c.join(',')})`))
            let trimmedPossibleColors = possibleColors.filter(color => {
              let hsl = color.toHsl()
              return hsl.l > .1 && hsl.l < .9 && hsl.s > .2
            })

            let rgbColor = trimmedPossibleColors.length == 0 ? possibleColors[0] : trimmedPossibleColors[0]
            let codeColor = rgbColor.isLight() ? 'black' : 'white'

            img.remove()
            let backgroundColor = rgbColor.toHex()
            let codeURL = `https://scannables.scdn.co/uri/plain/png/${backgroundColor}/${codeColor}/640/${vm.albumCodeCheck ? s.track.album.uri : s.track.uri}`
            tempSongs.push({cover: s.track.album.images[0].url, trackURI: s.track.uri, albumURI: s.track.album.uri, code: codeURL, color: rgbColor, key: i, id: s.track.id, albumID: s.track.album.id, artistNames: s.track.album.artists.map(a => a.name).join(','), albumName: s.track.album.name })
            if (tempSongs.length == initialSongs.length) {
              console.log('Done calculating')
              resolve(tempSongs)
            }
          })
        })

      });
    },

    processSongs(unprocessedSongs) {
      let vm = this;
      let process = (songs) => {
        if (vm.albumDupCheck && vm.aggressiveDupCheck) {
          songs = _.uniqBy(songs, v => v.color.toHex() + v.artistNames)//v.albumName.toLowerCase().split(' (')[0].split(' -')[0])
        }
        if (vm.albumDupCheck) {
          songs = _.uniqBy(songs, v => v.albumName.toLowerCase().split(' (')[0].split(' -')[0])
        }
        return songs
      }
      let newSongs = process(unprocessedSongs)

      if (this.oldSort != this.sortType) {
        newSongs = newSongs.sort(this.sortFunctions[this.sortType])
        //this.oldSort = this.sortType
      }
      return newSongs
    },
    removeTrack(key) {
      let idx = this.processed.findIndex(item => item !== null && item.key === key);
      //this.removed.push(key)
      if (idx == -1) {
        return
      }
      this.processed.splice(idx, 1)
      this.processed = this.arrangeFunctions[this.arrangeType](this.processed, this.colCount, this.rowCount)
    },
    processTracks() {
      let vm = this;
      this.processing = true
      this.raw = this.raw.filter(s => s.track !== null).filter(s => s.track.album.images.length >= 3)
      this.generateProcessedImages(this.raw, this.unprocessed)
      .then(unprocessedSongs => {
        console.log('UNPROCESSED =>', unprocessedSongs)
        vm.unprocessed = unprocessedSongs
        return vm.processSongs(unprocessedSongs)
      })
      .then((processedSongs) => {
        if (processedSongs.length != vm.processed.length) {
          vm.processed = processedSongs
        }
        console.log('PROCESSED =>', processedSongs)
        let selAsp = vm.aspects[vm.selectedAspect]
        vm.rowCount = vm.selectedFormat == 'Columns' ? Math.ceil(vm.processed.length / parseInt(vm.columns)) : Math.ceil(Math.sqrt(vm.processed.length / (selAsp.width * selAsp.height)) * selAsp.height)
        vm.colCount = vm.selectedFormat == 'Columns' ? parseInt(vm.columns) : Math.floor(Math.sqrt(vm.processed.length / (selAsp.width * selAsp.height)) * selAsp.width)
        vm.processed = vm.arrangeFunctions[vm.arrangeType](processedSongs, vm.colCount, vm.rowCount)
        vm.processing = false
      })
    }

  }
}
</script>
<style>
.v-progress-circular {
  transition: none !important;
}
</style>
