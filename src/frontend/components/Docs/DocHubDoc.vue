<template>
  <div>
    <v-alert v-if="!isReloading && error" icon="error" type="error">
      <h2>Ошибка!</h2>
      <div>Расположение: {{ path }}</div>
      <div>{{ error }}</div>
    </v-alert>
    <template v-if="!isReloading && !error">
      <component 
        v-bind:is="is"
        v-if="is"
        v-bind:inline="inline"
        v-bind:params="currentParams"
        v-bind:profile="profile"
        v-bind:path="currentPath"
        v-bind:get-content="getContentForPlugin"
        v-bind:to-print="isPrintVersion"
        v-bind:pull-data="pullData"
        v-bind:context-menu="contextMenu" />
      <template v-else>
        <v-alert v-if="profile && !isReloading" icon="warning">
          Неизвестный тип документа [{{ docType }}]
        </v-alert>
        <spinner v-else />
      </template>
    </template>
  </div>
</template>

<script>
  import { DocTypes } from '@front/components/Docs/enums/doc-types.enum';
  import AsyncApiComponent from '@front/components/Docs/DocAsyncApi.vue';
  import Empty from '@front/components/Controls/Empty.vue';
  import requests from '@front/helpers/requests';
  import datasets from '@front/helpers/datasets';
  import query from '@front/manifest/query';
  import uriTool from '@front/helpers/uri';

  import Swagger from './DocSwagger.vue';
  import Plantuml from './DocPlantUML.vue';
  import DocMarkdown from './DocMarkdown.vue';
  import DocTable from './DocTable.vue';
  import DocMermaid from './DocMermaid.vue';
  import DocNetwork from './DocNetwork.vue';
  import DocSmartants from './DocSmartAnts.vue';
  import Spinner from '@front/components/Controls/Spinner.vue';
  
  // Встроенные типы документов
  const inbuiltTypes = {
    [DocTypes.ASYNCAPI]: 'async-api-component',
    [DocTypes.OPENAPI]: 'swagger',
    [DocTypes.PLANTUML]: 'plantuml',
    [DocTypes.MARKDOWN]: 'doc-markdown',
    [DocTypes.TABLE]: 'doc-table',
    [DocTypes.MERMAID]: 'doc-mermaid',
    [DocTypes.NETWORK]: 'doc-network',
    [DocTypes.SMARTANTS]: 'doc-smartants'
  };

  
  export default {
    name: 'Document',
    components: {
      AsyncApiComponent,
      Plantuml,
      Swagger,
      DocMarkdown,
      DocTable,
      Empty,
      DocMermaid,
      DocNetwork,
      DocSmartants,
      Spinner
    },
    props: {
      path: {
        type: String,
        default: '$URL$'
      },
      inline: { type: Boolean, default: false },
      // Параметры передающиеся в запросы документа
      // Если undefined - берутся из URL
      params: { 
        type: Object,
        default: undefined
      },
      contextMenu: {
        type: Array,
        default() {
          return [];
        }
      }
    },
    data() {
      return {
        DocTypes,
        refresher: null,
        profile: null,
        error: null,
        currentPath : this.resolvePath(),
        currentParams: this.resolveParams(),
        dataProvider: datasets()
      };
    },
    computed: {
      is() {
        return inbuiltTypes[this.docType] 
          || (this.$store.state.plugins.documents[this.docType] && `plugin-doc-${this.docType}`)
          || null;
      },
      docType() {
        return (this.profile?.type || 'unknown').toLowerCase();
      },
      baseURI() {
        return uriTool.getBaseURIOfPath(this.currentPath);
      },
      isReloading() {
        return this.$store.state.isReloading;
      },
      isPrintVersion() {
        return this.$store.state.isPrintVersion;
      }
    },
    watch: {
      '$route'() {
        this.refresh();
      },
      params() {
        this.refresh();
      },
      isReloading() {
        this.refresh();
      }
    },
    mounted() {
      this.refresh();
    },
    methods: {
      // Обновляем контент документа
      refresh() {
        if (this.refresher) clearTimeout(this.refresher);
        this.refresher = setTimeout(() => {
          this.profile = null;
          const dateLakeId = `("${this.resolvePath().slice(1).split('/').join('"."')}")`;
          query.expression( query.getObject(dateLakeId), null, this.resolveParams())
            .evaluate()
            .then((profile) => {
              this.profile = Object.assign({ $base: this.path }, profile);
            })
            .catch((e) => {
              this.error = e.message;
            })
            .finally(() => {
              this.currentPath = this.resolvePath();
              this.currentParams = this.resolveParams();
              this.refresher = null;
            });

        }, 50);
      },
      resolveParams() {
        return this.params || this.$router.currentRoute.query || {};
      },  
      // Определяем текущий путь к профилю документа
      resolvePath() {
        if (this.path === '$URL$') return this.$router.history.current.path;
        return this.profile?.$base || this.path;
      },
      // Провайдер контента файлов для плагинов
      //  url - прямой или относительный URL к файлу
      getContentForPlugin(url) {
        return new Promise((success, reject) => {
          requests.request(url, this.baseURI)
            .then(success)
            .catch(reject);
        });
      },
      // API к озеру данных архитектуры
      //  expression - JSONata запрос или идентификатор ресурса
      //  self - значение переменной $self в запросе
      //  params - значение переменной $params в запросе
      //  context - контекст запроса (по умолчанию равен manifest)
      pullData(expression, self_, params, context) {
        if (!expression) {
          return this.dataProvider.releaseData(this.resolvePath(), params || this.params);
        } else
          return this.dataProvider.getData(context, { source: expression }, params);
          // return query.expression(expression || this.profile?.source, self_, params).evaluate(context);
      }
    }
  };
</script>
