<script setup lang="ts">
import { ref } from 'vue'
import exportScreenshot from '../assets/export.jpg'

const file = ref<File | null>(null)
const tracks = ref<any[]>([])
const playlists = ref<any[]>([])
const errors = ref<any[]>([])

type Playlist = {
  name: string
  trackIds: string[]
}

const parseXml = (xmlDoc: XMLDocument) => {
  parseTracks(xmlDoc)
  parsePlaylists(xmlDoc)
}

const parsePlaylists = (xmlDoc: XMLDocument) => {
  const playlistsElement = xmlDoc.getElementsByTagName("PLAYLISTS")[0]
  if (!playlistsElement) {
    errors.value = [...errors.value, 'Could not find PLAYLISTS tag in document']
    console.error('Could not find PLAYLISTS tag in document', xmlDoc)
    return
  }
  console.log('Parsing elements')

  playlists.value = parsePlaylistNodes(playlistsElement)
}

const totalPlaytimeSeconds = (trackIds: string[]): number => {
  let seconds = 0
  trackIds.forEach((trackId) => {
    let track = tracks.value.find((track) => track.TrackID === trackId)
    if (!track.TotalTime) {
      console.warn('TotalTime is missing from track', track)
      return
    }
    seconds = seconds + parseInt(track.TotalTime)
  })
  return seconds
}

const averageTrackPlaytimeSeconds = (trackIds: string[]): number => {
  let trackLengths: number[] = []
  trackIds.forEach((trackId) => {
    let track = tracks.value.find((track) => track.TrackID === trackId)
    if (!track.TotalTime) {
      console.warn('TotalTime is missing from track', track)
      return
    }
    trackLengths.push(parseInt(track.TotalTime))
  })
  if (trackLengths.length === 0) {
    return 0
  }
  return trackLengths.reduce((a, b) => a + b) / trackLengths.length
}

/**
 * Take the bpm and time, dismiss the first and last 16 bars
 */
const totalSettimeSeconds = (trackIds: string[]): number => {
  let seconds = 0
  trackIds.forEach((trackId) => {
    let track = tracks.value.find((track) => track.TrackID === trackId)
    if (!track.AverageBpm) {
      console.warn('AverageBpm is missing from track', track)
      return
    }
    if (!track.TotalTime) {
      console.warn('TotalTime is missing from track', track)
      return
    }
    let sixteenBarsInBeats = 16 * 4
    let trackInSeconds = parseInt(track.TotalTime)
    let beatsPerSecond = 60 / parseFloat(track.AverageBpm)
    let sixteenBarsInSeconds = beatsPerSecond * sixteenBarsInBeats

    // Take sixteenBarsInSeconds off start and end
    trackInSeconds = trackInSeconds - (sixteenBarsInSeconds * 2)

    seconds = seconds + parseInt(trackInSeconds)
    // TotalTime, AverageBpm, Tonality
  })
  return seconds
}

const parsePlaylistNodes = (playlistsElement) => {
  const playlists: Playlist[] = []

  function traverseNodes(node: Element): void {
    const nodeName = node.getAttribute('Name')
    const nodeType = node.getAttribute('Type')

    // Only collect playlists that have track entries (Type="1")
    if (nodeType === '1' && nodeName) {
      const trackElements = node.getElementsByTagName('TRACK')
      const trackIds: string[] = []

      for (let i = 0; i < trackElements.length; i++) {
        const trackId = trackElements[i].getAttribute('Key')
        if (trackId) {
          trackIds.push(trackId)
        }
      }

      if (trackIds.length === 0) {
        console.warn(nodeName, 'does not contain any tracks')
        return
      }

      playlists.push({
        name: nodeName,
        trackIds,
        totalPlaytime: totalPlaytimeSeconds(trackIds),
        totalSettime: totalSettimeSeconds(trackIds),
        averageTrackPlaytimeSeconds: averageTrackPlaytimeSeconds(trackIds),
      })
    }

    // Recursively handle child NODEs
    const childNodes = node.getElementsByTagName('NODE');
    for (let i = 0; i < childNodes.length; i++) {
      // Ensure direct children only to avoid duplicate processing
      if (childNodes[i].parentElement === node) {
        traverseNodes(childNodes[i])
      }
    }
  }

  const rootNodes = playlistsElement.getElementsByTagName('NODE')
  for (let i = 0; i < rootNodes.length; i++) {
    if (rootNodes[i].parentElement === playlistsElement) {
      traverseNodes(rootNodes[i]);
    }
  }

  return playlists
}

const parseTracks = (xmlDoc: XMLDocument) => {
  const collectionElement = xmlDoc.getElementsByTagName("COLLECTION")[0]
  const trackElements = collectionElement.getElementsByTagName("TRACK")

  const tempTracks: any[] = []

  // Pop all the tracks into an array to make them easier to find later
  for (let i = 0; i < trackElements.length; i++) {
    const track = trackElements[i]
    const attrs: any = {}

    // @TODO 2025-04-12 Map this into defined properies with fallbacks
    for (let j = 0; j < track.attributes.length; j++) {
      const attr = track.attributes[j]
      attrs[attr.name] = attr.value
    }

    tempTracks.push(attrs)
  }

  tracks.value = tempTracks
}

const uploadFile = (event: Event) => {
  console.log('Uploading file')
  const target = event.target as HTMLInputElement
  if (target.files && target.files[0]) {
    file.value = target.files[0]

    const reader = new FileReader()
    reader.readAsText(file.value)

    reader.onload = () => {
      const xmlString = reader.result as string
      const parser = new DOMParser()
      const xmlDoc = parser.parseFromString(xmlString, "text/xml")
      parseXml(xmlDoc)
    }
  }
}

const secondsToMinutes = (seconds: number) => {
  return new Date(seconds * 1000)
    .toISOString()
    .slice(14, 19)
}
const secondsToHours = (seconds: number) => {
  return new Date(seconds * 1000)
    .toISOString()
    .slice(11, 19)
}
</script>

<template>
  <p>
    <input type="file" accept="text/xml" @change="uploadFile">
  </p>

  <div v-if="playlists.length === 0">
    <h2>Export Collection in xml format</h2>
    <p><img :src="exportScreenshot" width="300px"></p>
  </div>

  <table v-if="playlists.length > 0">
    <thead>
      <tr>
        <th class="row-title">Playlist</th>
        <th>Tracks</th>
        <th>Average</th>
        <th>Playtime</th>
        <th>Settime</th>
        <th>Shrinkage</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="(playlist, playlistIndex) in playlists" :key="`playlist${playlistIndex}`">
        <th class="row-title">{{ playlist.name }}</th>
        <td class="time">{{ playlist.trackIds.length }}</td>
        <td class="time">{{ secondsToMinutes(playlist.averageTrackPlaytimeSeconds) }}</td>
        <td class="time">{{ secondsToHours(playlist.totalPlaytime) }}</td>
        <td class="time">{{ secondsToHours(playlist.totalSettime) }}</td>
        <td class="time">{{ parseInt((playlist.totalSettime / playlist.totalPlaytime) * 100) }}%</td>
      </tr>
    </tbody>
  </table>

  <footer>
    <div>
      <p>🧮 The calculations assume 16 bars in 4/4 are overlapping on each song</p>
      <p>⚠️ These are guesses and this website does not accept any liability for death or injury caused</p>
      <p>☁️ This is an in-browser app, none of your data is uploaded to any clouds</p>
    </div>
    <div>
      <p>™️ This website is not affiliated with rekordbox, AlphaTheta or Pioneer DJ</p>
      <p>🏳️‍⚧️ 🇵🇸 None of us are free until all of us are free</p>
      <p>&copy; 2025 Zoe Edwards</p>
    </div>
  </footer>
</template>

<style scoped>
footer {
  margin-top: 2rem;
  font-size: .8rem;
  color: #888;
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: calc(1rem);
}
.row-title {
  text-align: left;
}
.time {
  font-variant-numeric: tabular-nums;
  text-align: right;
}
td, th {
  padding: .3rem .4rem;
}
</style>
