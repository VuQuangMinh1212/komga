<template>
  <v-container fluid class="pa-6">
    <v-alert type="info" text class="body-2">{{ $t('book_upload.info') }}</v-alert>

    <v-row>
      <v-col cols="12" sm="4">
        <v-select
          v-model="libraryId"
          :items="libraries"
          item-text="name"
          item-value="id"
          :label="$t('book_upload.library')"
          :disabled="running"
        />
      </v-col>

      <v-col cols="12" sm="8">
        <v-combobox
          v-model="series"
          :items="seriesFolders"
          :label="$t('book_upload.series')"
          :hint="seriesHint"
          persistent-hint
          clearable
          :disabled="running"
          @update:search-input="onType"
        />
      </v-col>
    </v-row>

    <v-file-input
      v-model="picked"
      multiple
      show-size
      hide-details
      class="mb-4"
      accept=".cbz,.cbr,.cb7,.cbt,.zip,.rar,.7z,.pdf,.epub"
      :label="$t('book_upload.files')"
      @change="addFiles"
    />

    <v-simple-table v-if="items.length > 0" dense class="mb-4">
      <tbody>
      <tr v-for="it in items" :key="it.id">
        <td style="width: 36px">
          <v-icon :color="statusColor(it)">{{ statusIcon(it) }}</v-icon>
        </td>
        <td class="text-break">{{ it.file.name }}</td>
        <td class="text-no-wrap">{{ size(it.file.size) }}</td>
        <td style="min-width: 180px">
          <v-progress-linear v-if="it.status === 'uploading'" :value="it.progress" height="16">
            <small>{{ it.progress }}%</small>
          </v-progress-linear>
          <span v-else :class="statusColor(it) + '--text'" :title="it.error">{{ statusText(it) }}</span>
        </td>
        <td class="text-no-wrap text-right">
          <v-btn v-if="it.status === 'error'" small text color="primary" @click="retry(it)">
            {{ $t('book_upload.retry') }}
          </v-btn>
          <v-btn icon small :disabled="it.status === 'uploading'" @click="remove(it)">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </td>
      </tr>
      </tbody>
    </v-simple-table>

    <div v-if="items.length > 0" class="body-2 mb-4">{{ summary }}</div>

    <v-btn color="primary" class="mr-2" :disabled="!canStart" :loading="running" @click="start">
      {{ $t('book_upload.button_upload') }}
    </v-btn>
    <v-btn v-if="running" class="mr-2" @click="cancel">{{ $t('book_upload.cancel') }}</v-btn>
    <v-btn v-if="hasFinished && !running" text @click="clearFinished">{{ $t('book_upload.clear_finished') }}</v-btn>
  </v-container>
</template>

<script lang="ts">
import Vue from 'vue'
import {LibraryDto} from '@/types/komga-libraries'
import {ERROR, NOTIFICATION} from '@/types/events'
import {convertErrorCodes} from '@/functions/error-codes'
import {SeriesSearch} from '@/types/komga-search'

type Status = 'pending' | 'uploading' | 'done' | 'exists' | 'error'

interface QueueItem {
  id: number
  file: File
  status: Status
  progress: number
  error: string
}

export default Vue.extend({
  name: 'UploadBooks',
  data: () => ({
    libraryId: '' as string,
    series: '' as string,
    picked: [] as File[],
    items: [] as QueueItem[],
    seriesFolders: [] as string[],
    running: false,
    cancelled: false,
    abort: null as AbortController | null,
    nextId: 1,
  }),
  computed: {
    libraries(): LibraryDto[] {
      return this.$store.state.komgaLibraries.libraries
    },
    canStart(): boolean {
      return !this.running && !!this.libraryId && !!(this.series || '').trim() &&
        this.items.some(i => i.status === 'pending')
    },
    hasFinished(): boolean {
      return this.items.some(i => i.status === 'done' || i.status === 'exists')
    },
    summary(): string {
      const n = (s: Status) => this.items.filter(i => i.status === s).length
      return this.$t('book_upload.summary', {
        done: n('done'), skipped: n('exists'), failed: n('error'), waiting: n('pending') + n('uploading'),
      }).toString()
    },
    seriesHint(): string {
      const name = (this.series || '').trim()
      if (name && !this.seriesFolders.includes(name)) return this.$t('book_upload.series_hint_new', {name}).toString()
      return this.$t('book_upload.series_hint').toString()
    },
  },
  watch: {
    libraryId: 'loadSeriesFolders',
  },
  mounted() {
    if (this.libraries.length > 0) this.libraryId = this.libraries[0].id
  },
  beforeDestroy() {
    // leaving the page stops the queue
    this.cancelled = true
    if (this.abort) this.abort.abort()
  },
  methods: {
    onType(text: string | null) {
      // a typed name counts right away, no need to press Enter
      if (text) this.series = text
    },
    async loadSeriesFolders() {
      // series folder = last segment of the series path, so uploads land in the existing folder
      const all = (await this.$komgaSeries.getSeriesList({} as SeriesSearch, {unpaged: true})).content
      this.seriesFolders = [...new Set(all
        .filter(s => s.libraryId === this.libraryId && !s.deleted) // skip series in Komga's trash
        .map(s => s.url.replace(/[\\/]+$/, '').split(/[\\/]/).pop() as string))].sort()
    },

    // ---- queue ----
    addFiles(files: File[]) {
      for (const f of files || []) {
        const dup = this.items.some(i => i.file.name === f.name && i.file.size === f.size && i.status !== 'done')
        if (!dup) this.items.push({id: this.nextId++, file: f, status: 'pending', progress: 0, error: ''})
      }
      this.$nextTick(() => this.picked = []) // ready for the next selection
      if (this.running) return // the running queue picks new files up by itself
    },
    remove(it: QueueItem) {
      this.items = this.items.filter(i => i !== it)
    },
    retry(it: QueueItem) {
      it.status = 'pending'
      it.error = ''
      it.progress = 0
      if (!this.running && this.canStart) this.start()
    },
    clearFinished() {
      this.items = this.items.filter(i => i.status !== 'done' && i.status !== 'exists')
    },
    cancel() {
      this.cancelled = true
      if (this.abort) this.abort.abort()
    },
    async start() {
      if (!this.canStart) return
      this.running = true
      this.cancelled = false
      // fixed for the whole run
      const libraryId = this.libraryId
      const series = this.series.trim()
      let uploaded = 0

      // one file per request, one at a time (keeps each request small and below the proxy's body limit)
      let next: QueueItem | undefined
      while (!this.cancelled && (next = this.items.find(i => i.status === 'pending'))) {
        const item = next
        item.status = 'uploading'
        item.progress = 0
        this.abort = new AbortController()
        try {
          await this.$komgaLibraries.uploadBooks(libraryId, series, [item.file], p => item.progress = p,
            {scan: false, signal: this.abort.signal})
          item.status = 'done'
          item.progress = 100
          uploaded++
        } catch (e) {
          if (e.cancelled) {
            item.status = 'pending'
            break
          } else if (e.status === 409) {
            item.status = 'exists'
          } else {
            item.status = 'error'
            item.error = convertErrorCodes(e.message)
          }
        }
      }
      this.abort = null
      this.running = false
      this.cancelled = false

      if (uploaded > 0) {
        try {
          // a single scan once the queue is finished
          const library = this.libraries.find(l => l.id === libraryId)
          if (library) await this.$komgaLibraries.scanLibrary(library)
          this.$eventHub.$emit(NOTIFICATION, {message: this.$t('book_upload.done', {count: uploaded}).toString()})
        } catch (e) {
          this.$eventHub.$emit(ERROR, {message: convertErrorCodes(e.message)} as ErrorEvent)
        }
        await this.loadSeriesFolders()
      }
    },

    // ---- display ----
    size(bytes: number): string {
      return bytes >= 1048576 ? `${(bytes / 1048576).toFixed(1)} MB` : `${Math.max(1, Math.round(bytes / 1024))} KB`
    },
    statusIcon(it: QueueItem): string {
      return {
        pending: 'mdi-clock-outline', uploading: 'mdi-upload', done: 'mdi-check-circle',
        exists: 'mdi-skip-next-circle-outline', error: 'mdi-alert-circle',
      }[it.status]
    },
    statusColor(it: QueueItem): string {
      return {pending: 'grey', uploading: 'primary', done: 'success', exists: 'warning', error: 'error'}[it.status]
    },
    statusText(it: QueueItem): string {
      return it.status === 'error' && it.error ? it.error : this.$t(`book_upload.status.${it.status}`).toString()
    },
  },
})
</script>
