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
          @update:search-input="onType"
        />
      </v-col>
    </v-row>

    <v-file-input
      v-model="files"
      multiple
      chips
      show-size
      accept=".cbz,.cbr,.cb7,.cbt,.zip,.rar,.7z,.pdf,.epub"
      :label="$t('book_upload.files')"
    />

    <v-progress-linear v-if="uploading" :value="progress" height="20" class="mb-4">
      <strong>{{ progress }}%</strong>
    </v-progress-linear>

    <v-btn color="primary" :disabled="!canUpload" :loading="uploading" @click="upload">
      {{ $t('book_upload.button_upload') }}
    </v-btn>
  </v-container>
</template>

<script lang="ts">
import Vue from 'vue'
import {LibraryDto} from '@/types/komga-libraries'
import {ERROR, NOTIFICATION} from '@/types/events'
import {convertErrorCodes} from '@/functions/error-codes'
import {SeriesSearch} from '@/types/komga-search'

export default Vue.extend({
  name: 'UploadBooks',
  data: () => ({
    libraryId: '' as string,
    series: '' as string,
    files: [] as File[],
    seriesFolders: [] as string[],
    uploading: false,
    progress: 0,
  }),
  computed: {
    libraries(): LibraryDto[] {
      return this.$store.state.komgaLibraries.libraries
    },
    canUpload(): boolean {
      return !!this.libraryId && !!(this.series || '').trim() && this.files.length > 0 && !this.uploading
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
    async upload() {
      this.uploading = true
      this.progress = 0
      try {
        await this.$komgaLibraries.uploadBooks(this.libraryId, this.series.trim(), this.files, p => this.progress = p)
        this.$eventHub.$emit(NOTIFICATION, {message: this.$t('book_upload.done', {count: this.files.length}).toString()})
        this.files = []
        await this.loadSeriesFolders()
      } catch (e) {
        this.$eventHub.$emit(ERROR, {message: convertErrorCodes(e.message)} as ErrorEvent)
      } finally {
        this.uploading = false
      }
    },
  },
})
</script>
