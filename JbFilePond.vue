<template>
  <file-pond
    ref="filepond"
    v-on="this.$listeners"
    v-bind="this.$attrs"
    :server="server"
    :instant-upload="instantUpload"
    :allow-multiple="multiple || maxFiles > 1"
    :allow-file-type-validation="permitirValidacaoPorTipo"
    :acceptedFileTypes="acceptedFileTypes"
    
    :include-styling="false"
    imagePreviewMaxHeight="128"
    :label-idle="label"
    :label-file-type-not-allowed="labelFileTypeNotAllowed"
    :label-file-processing-complete="labelFileProcessingComplete"
    :label-file-processing="labelFileProcessing"
    :label-tap-to-undo="labelTapToUndo"
    :label-file-load-error="labelFileLoadError"
    :label-tap-to-retry="labelTapToRetry"
    :label-file-waiting-for-size="labelFileWaitingForSize"
    :file-validate-type-label-expected-types="
      fileValidateTypeLabelExpectedTypes
    "
    @init="init"
    
  />
</template>

<script>
import axios from 'axios'

import vueFilePond from 'vue-filepond'
import 'filepond/dist/filepond.min.css'

import FilePondPluginFileValidateType from 'filepond-plugin-file-validate-type'

import FilePondPluginImagePreview from 'filepond-plugin-image-preview'
import 'filepond-plugin-image-preview/dist/filepond-plugin-image-preview.css'

const FilePond = vueFilePond(
  FilePondPluginFileValidateType,
  FilePondPluginImagePreview
)

export default {
  components: { FilePond },
  props: {
    csrf: String,

    urlBase: { type: String, default: '/' },
    url: { type: String, default: 'arquivos' },
    urlParteDeletar: { type: String, default: 'deletar' },
    ativarHttp: { type: Boolean, default: true },

    jwtToken: String,
    storagePath: { type: String, default: 'storage/app' },

    // FILEPOND
    acceptedFileTypes: Array,
    instantUpload: { type: Boolean, default: true },
    maxFiles: { type: Number, default: null },
    multiple: Boolean,

    label: {
      type: String,
      default: '<b>Clique aqui</b> ou arraste o arquivo pra cá'
    },
    labelFileTypeNotAllowed: {
      type: String,
      default: 'Tipo de arquivo não permitido'
    },
    labelFileProcessingComplete: { type: String, default: 'Arquivo Carregado' },
    labelFileProcessing: { type: String, default: 'Carregando...' },
    labelTapToUndo: {
      type: String,
      default: 'Clique no X para carregar novo arquivo'
    },
    labelFileLoadError: { type: String, default: 'Erro ao carregar' },
    labelTapToRetry: { type: String, default: 'Clique para repetir' },
    labelFileWaitingForSize: { type: String, default: '? b' },
    fileValidateTypeLabelExpectedTypes: {
      type: String,
      default: 'Tente: {allButLastType} ou {lastType}'
    }
  },
  data() {
    return {
      vmodel: {},
      url_base: this.urlBase,
      url_data: this.url,
      server: {
        process: (fieldName, file, metadata, load, error, progress, abort) =>
          this.process(fieldName, file, metadata, load, error, progress, abort),
        load: (source, load, error, progress, abort, headers) =>
          this.load(source, load, error, progress, abort, headers),
        revert: (response_result, load, error) =>
          this.revert(response_result, load, error),
        remove: (source, load, error) => this.remove(source, load, error)
      }
    }
  },
  computed: {
    url_process() {
      return `${this.url_base}/${this.url_data}`
    },
    url_deletar() {
      return `${this.url_base}/${this.url_data}/${this.urlParteDeletar}`
    },
    permitirValidacaoPorTipo(){
      return true || !!this.$attrs['accepted-file-types']
    }
  },
  watch: {
    value(v) {
      if (!v) this.vmodel = {}
    }
  },
  methods: {
    init() {
      console.log('FilePond foi iniciado e adicionado em `this.$refs.filepond`')
      
    },
    process(fieldName, file, metadata, load, error, progress, abort) {
      const data = Object.assign(new FormData(), this.value)
      data.append(fieldName, file, file.name)

      const CancelToken = axios.CancelToken
      const source = CancelToken.source()

      const config = {
        onUploadProgress: e => {
          progress(e.lengthComputable, e.loaded, e.total)
        },
        headers: { 'Content-Type': 'multipart/form-data' },
        cancelToken: source.token
      }

      axios
        .post(this.url_process, data, config)
        .then(response => {
          load(response)
          this.setVmodel('process', false, response)
          this.$emit('success', response)
          this.$emit('complete', response)
        })
        .catch(responseError => {
          error('oh no')
          this.setVmodel('process', true, responseError.response)
          this.$emit('complete', responseError)
          this.$emit('error', responseError)
        })

      return {
        abort: () => {
          source.cancel('Operation canceled by the user.')
          abort()
        }
      }
    },
    load(source, load, error, progress, abort, headers) {
      source = `${this.storagePath}/${source}`

      axios
        .post(source)
        .then(response => {
          return response.blob()
        })
        .then(response => {
          load(response)
          this.setVmodel('load', false, response)
          this.$emit('success', response)
          this.$emit('complete', response)
        })
        .catch(responseError => {
          error('oh no')
          this.setVmodel('process', true, responseError.response)
          this.$emit('error', responseError)
          this.$emit('complete', responseError)
        })

      return {
        abort: () => {
          abort()
        }
      }
    },
    revert(response_result, load, error) {
      if (!response_result) {
        return
      }

      const data = Object.assign(new FormData(), this.value)
      data.append('file', response_result)

      axios
        .post(this.url_deletar, data)
        .then(response => {
          load(response.data)
          this.setVmodel('revert', false, response)
          this.$emit('success', response)
          this.$emit('complete', response)
        })
        .catch(responseError => {
          error('oh no')
          this.setVmodel('revert', true, responseError.response)
          this.$emit('error', responseError)
          this.$emit('complete', responseError)
        })
    },
    remove(source, load, error) {
      // Should somehow send `source` to server so server can remove the file with this source
      let source_temp = source.split('/')
      let filename = source_temp.pop()
      source = source_temp.join('/')

      let dados = { file_id: filename }

      if (this.ativarHttp) {
        this.server.revert(dados, load, error)
      } else {
        load()
      }
    },
    criarRequest(nova_url) {
      let url = nova_url || this.url_data
      let request = new XMLHttpRequest()
      request.open('POST', `${this.url_base}/${url}`)
      if (this.csrf) {
        request.setRequestHeader('X-CSRF-TOKEN', this.csrf)
      }
      if (this.jwtToken) {
        request.setRequestHeader('Authorization', `Bearer ${this.jwtToken}`)
      }
      return request
    },
    setVmodel(origem, serverError, response) {
      if (!response) {
        return
      }

      let data = response.data
      let vmodel = {
        error: serverError,
        status: response.status,
        statusText: response.statusText,
        data: data
      }

      let file_id = data.id || Date.now()

      if (origem == 'load') {
        console.log('load aki')

        let url = decodeURIComponent(data.url)
        file_id = url.split('/').pop()
        let caminho_completo = url.substr(url.search(this.storagePath))

        let respostaPadrao = this.getVmodelRespostaPadrao(
          false,
          file_id,
          caminho_completo,
          file_id,
          origem
        )
        // let inicio = caminho_completo.search(this.storagePath)

        Object.assign(response, respostaPadrao)
        response.serverError = false
      }

      this.$set(this.vmodel, file_id, vmodel)
      this.emit()
    },
    emit() {
      let vmodel = this.vmodel

      if (!this.$refs.filepond.allowMultiple) {
        let index = Object.keys(vmodel).pop()
        vmodel = vmodel[index]
      }

      this.$emit('input', vmodel)
    }
  }
}
</script>

<style>
.filepond--drop-label label,
.filepond--file-action-button {
  cursor: pointer;
  font-size: 12px;
}

/* the background color of the file and file panel (used when dropping an image) */
.filepond--item-panel {
  background-color: #369763;
}

/* busy state color */
[data-filepond-item-state*='busy'] .filepond--item-panel {
  background-color: #64605e !important;
}

/* error state color */
[data-filepond-item-state*='error'] .filepond--item-panel,
[data-filepond-item-state*='invalid'] .filepond--item-panel {
  background-color: #c44e47 !important;
}
</style>
