<template>

    <file-pond
        ref="pond"
        :id="id"

        :name="name"
        :allow-multiple="multiple || maxFiles>1"
        :max-files="maxFiles"
        :max-parallel-uploads="maxParallelUploads"

        :server="server"
        :files="files"
        :instant-upload="instantUpload"


        :label-idle="label"
        :label-file-type-not-allowed="labelFileTypeNotAllowed"
        :label-file-processing-complete="labelFileProcessingComplete"
        :label-file-processing="labelFileProcessing"
        :label-tap-to-undo="labelTapToUndo"
        :label-file-load-error="labelFileLoadError"
        :label-tap-to-retry="labelTapToRetry"
        :label-file-waiting-for-size="labelFileWaitingForSize"

        :allow-file-type-validation="this.tiposArquivos && this.tiposArquivos.length > 0"
        :accepted-file-types="tiposArquivos"
        file-validate-type-label-expected-types="Tente: {allButLastType} ou {lastType}"

        :allow-image-preview="allowImagePreview"
        imagePreviewMaxHeight="128"

        :include-styling="false"

        @init="init"

    />

</template>

<script>

import vueFilePond from 'vue-filepond';
import 'filepond/dist/filepond.min.css';

import FilePondPluginFileValidateType from 'filepond-plugin-file-validate-type';

import FilePondPluginImagePreview from 'filepond-plugin-image-preview';
import 'filepond-plugin-image-preview/dist/filepond-plugin-image-preview.css';

const FilePond = vueFilePond(FilePondPluginFileValidateType, FilePondPluginImagePreview);

export default {
    components: { FilePond },
    props:{
        value:Object, id:String,
        csrf:String, height:String,
        name:String,

        files:Array,

        multiple:Boolean,
        maxFiles:{type:Number, default:null},
        maxParallelUploads:{type:Number, default:100},
        instantUpload:{type:Boolean, default:true},

        tiposArquivos:String,

        label:{type:String, default:'<b>Clique aqui</b> ou arraste o arquivo pra cá'},
        labelFileTypeNotAllowed:{type:String, default:'Tipo de arquivo não permitido'},
        labelFileProcessingComplete:{type:String, default:'Arquivo Carregado'},
        labelFileProcessing:{type:String, default:'Carregando...'},
        labelTapToUndo:{type:String, default:'Clique para carregar novo arquivo'},
        labelFileLoadError:{type:String, default:'Erro ao carregar'},
        labelTapToRetry:{type:String, default:'Clique para repetir'},
        labelFileWaitingForSize:{type:String, default:'? b'},

        allowImagePreview:Boolean,
        allowFileTypeValidation:Boolean,

        urlBase:{type:String, default:'/'},
        url:{type:String, default:'arquivos'},
        ativarHttp:{type:Boolean, default:true},
    },
    data: function() {
        return {
            vModel:{},
            url_base: this.urlBase,
            url_data: this.url,
            server: {
                process:(fieldName, file, metadata, load, error, progress, abort) => {

                    // fieldName is the name of the input field and file is the actual file object to send
                    const formData = Object.assign(new FormData(), this.value);
                    formData.append(fieldName, file, file.name);

                    const request = new XMLHttpRequest();
                    request.open('POST', this.url_base + this.url_data);
                    request.setRequestHeader('X-CSRF-TOKEN',this.csrf)

                    // Should call the progress method to update the progress to 100% before calling load
                    // Setting computable to false switches the loading indicator to infinite mode
                    request.upload.onprogress = (e) => {
                        progress(e.lengthComputable, e.loaded, e.total);
                    };

                    // Should call the load method when done and pass the returned server file id
                    // this server file id is then used later on when reverting or restoring a file
                    // so your server knows which file to return without exposing that info to the client

                    request.onload = () => {
                        let response = JSON.parse(request.response)
                        let temErro = {}.hasOwnProperty.call(response, 'erro') ? response.erro : false

                        if (request.status >= 200 && request.status < 300) {
                            // the load method accepts either a string (id) or an object

                            if(temErro){
                                error('oh no');
                            }
                            else {
                                load(request.responseText);
                            }

                            this.setVModel('process',false, JSON.parse(request.response), formData);
                        }
                        else {
                            // Can call the error method if something is wrong, should exit after
                            error('oh no');
                            this.setVModel('process',true, request, formData);
                        }

                    };

                    request.send(formData);

                    // Should expose an abort method so the request can be cancelled
                    return {
                        abort: () => {
                            // This function is entered if the user has tapped the cancel button
                            request.abort();

                            // Let FilePond know the request has been cancelled
                            abort();
                        }
                    };
                },
                load: (source, load, error, progress, abort, headers) => {

                    let source_correcao = 'storage/app/'+source

                    let load_ok = false;
                    let response = null;
                    fetch(source_correcao)
                    .then(res => {
                        response = res;
                        load_ok = res.ok && res.status >= 200 && res.status < 300
                        return res.blob()
                    })
                    .then( promisse => {
                        error('oh my goodness');
                        if(load_ok){
                            load(promisse)
                            this.setVModel('load',false, response, {});
                        }
                    });

                    // Should expose an abort method so the request can be cancelled
                    return {
                        abort: () => {
                            // User tapped cancel, abort our ongoing actions here

                            // Let FilePond know the request has been cancelled
                            abort();
                        }
                    };
                },
                revert: (response, load, error) => {

                    if(typeof response=='string'){
                        response = JSON.parse(response)
                    }
                    // Should remove the earlier created temp file here
                    const request = new XMLHttpRequest();
                    request.open('POST', this.url_base + this.url_data + '/deletar');
                    request.setRequestHeader('X-CSRF-TOKEN',this.csrf)

                    if(this.value){
                        let obj = {};
                        if( this.$refs.pond.allowMultiple){
                            obj = this.value[response.filepond.key]
                        }
                        else {
                            obj = this.value
                        }


                        if(obj.response.erro){
                            load();
                            return;
                        }
                    }

                    const formData = Object.assign(new FormData(), this.value);
                    formData.append('nome', response.dados.nomeinterno);
                    formData.append('endereco', response.dados.endereco);

                    request.onload = () => {
                        let response = JSON.parse(request.response)

                        let temErro = {}.hasOwnProperty.call(response, 'erro') ? response.erro : false

                        if (request.status >= 200 && request.status < 300) {
                            if(temErro){
                                // o servidor processou a requisição mas ocorreu algum erro durante a processo
                                error('oh no');
                            }
                            else {
                                load(request.responseText);
                            }

                            this.setVModel('revert',false, JSON.parse(request.response), formData);
                        }
                        else {
                            // erro que impediu a requisição (servidor nao processou)
                            error('oh no');
                            this.setVModel('revert',true, request, formData);
                        }
                    };

                    request.send(formData)

                },
                remove: (source, load, error) => {
                    // Should somehow send `source` to server so server can remove the file with this source
                    let source_temp = source.split('/')
                    let filename = source_temp.pop()
                    source = source_temp.join('/')

                    let dados = {
                        filepond:{key:filename, acao:'remove'},
                        erro:false,
                        dados:{
                            nomeinterno:filename,
                            endereco:source
                        }
                    }

                    if(this.ativarHttp){
                        this.server.revert(dados,load,error)
                    }
                    else {
                        load()
                    }

                }
            },

        };
    },
    watch:{
        value(v){
            if(! v) this.vModel = {}
        }
    },
    methods: {
        getVModelRespostaPadrao(erro, nomeinterno, caminho_completo, key, acao){
            return {
                erro: erro||false,
                dados: {nomeinterno:nomeinterno||null, caminho_completo:caminho_completo||null},
                filepond: {key:key||null, acao:acao||null},
            }
        },
        handleFilePondInit() {
            console.log('FilePond has initialized');
            // FilePond instance methods are available on `this.$refs.pond`
        },
        setVModel(origem, serverError, responseJson, formData){
            let obj = { serverError:serverError, response: responseJson, dados: formData }
            let key = null

            if(origem=='process'){
                key = responseJson.filepond.key
                this.$set(this.vModel, key, obj)
            }
            else if(origem=='revert'){
                key = responseJson.filepond.key
                this.$set(this.vModel, key, null)
            }
            else if(origem=='load'){
                let url = decodeURIComponent(responseJson.url)
                key = url.split('/').pop()
                let caminho_completo = url.substr(url.search('storage/app')+12)

                let respostaPadrao = this.getVModelRespostaPadrao(false,key,caminho_completo,key,origem)
                let inicio = caminho_completo.search('storage/app') +12

                Object.assign(responseJson, respostaPadrao)
                responseJson.serverError = false

                this.$set(this.vModel, key, obj)
            }

            this.emitVModel();
        },
        emitVModel(){

            let vmodel = this.vModel

            if( ! this.$refs.pond.allowMultiple){
                let index = Object.keys(vmodel).pop();
                vmodel = vmodel[index];
            }

            this.$emit('input', vmodel)
        }
    },
};
</script>

<style>
.filepond--drop-label label, .filepond--file-action-button {
    cursor: pointer;
}


/* the background color of the file and file panel (used when dropping an image) */
.filepond--item-panel {
    background-color: #369763
}

/* busy state color */
[data-filepond-item-state*=busy] .filepond--item-panel {
    background-color: #64605e !important;
}

/* error state color */
[data-filepond-item-state*=error] .filepond--item-panel,
[data-filepond-item-state*=invalid] .filepond--item-panel {
    background-color: #c44e47 !important;
}


</style>
