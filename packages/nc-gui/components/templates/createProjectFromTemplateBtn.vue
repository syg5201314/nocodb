<template>
  <div>
    <v-btn
      :loading="projectCreation"
      :disabled="projectCreation"
      class="primary nc-btn-use-template"
      x-large
      @click="useTemplate('rest')"
    >
      <slot>Use template</slot>
    </v-btn>
  </div>
</template>

<script>
import { SqlUiFactory } from 'nocodb-sdk'
import colors from '~/mixins/colors'

export default {
  name: 'CreateProjectFromTemplateBtn',
  mixins: [colors],
  props: {
    excelImport: Boolean,
    loading: Boolean,
    importToProject: Boolean,
    templateData: [Array, Object],
    importData: [Array, Object],
    valid: {
      default: true,
      type: Boolean
    },
    validationErrorMsg: {
      default: 'Please fill all the required values',
      type: String
    },
    createGqlText: {
      default: 'Create GQL Project',
      type: String
    },
    createRestText: {
      default: 'Create REST Project',
      type: String
    }
  },
  data() {
    return {
      localTemplateData: null,
      projectCreation: false,
      tableCreation: false,
      loaderMessagesIndex: 0,
      loaderMessages: [
        'Setting up new database configs',
        'Inferring database schema',
        'Generating APIs.',
        'Generating APIs..',
        'Generating APIs...',
        'Generating APIs....',
        'Please wait',
        'Please wait.',
        'Please wait..',
        'Please wait...',
        'Please wait..',
        'Please wait.',
        'Please wait',
        'Please wait.',
        'Please wait..',
        'Please wait...',
        'Please wait..',
        'Please wait.',
        'Please wait..',
        'Please wait...'
      ]
    }
  },
  watch: {
    templateData: {
      deep: true,
      handler(data) {
        this.localTemplateData = JSON.parse(JSON.stringify(data))
      }
    }
  },
  created() {
    this.localTemplateData = JSON.parse(JSON.stringify(this.templateData))
  },

  methods: {
    async useTemplate(projectType) {
      if (!this.valid) {
        return this.$toast.error(this.validationErrorMsg).goAway(3000)
      }

      // this.$emit('useTemplate', type)
      // this.projectCreation = true
      let interv
      try {
        interv = setInterval(() => {
          this.loaderMessagesIndex = this.loaderMessagesIndex < this.loaderMessages.length - 1 ? this.loaderMessagesIndex + 1 : 6
          this.$store.commit('loader/MutMessage', this.loaderMessages[this.loaderMessagesIndex])
        }, 1000)

        let project

        // Not available now
        if (this.importToProject) {
          // this.$store.commit('loader/MutMessage', 'Importing excel template')

          // const res = await this.$store.dispatch('sqlMgr/ActSqlOp', [{
          //   // todo: extract based on active
          //   dbAlias: 'db', // this.nodes.dbAlias,
          //   env: '_noco'
          // }, 'xcModelsCreateFromTemplate', {
          //   template: this.templateData
          // }])

          // if (res && res.tables && res.tables.length) {
          //   this.$toast.success(`Imported ${res.tables.length} tables successfully`).goAway(3000)
          // } else {
          //   this.$toast.success('Template imported successfully').goAway(3000)
          // }

          // projectId = this.$route.params.project_id
          // prefix = this.$store.getters['project/GtrProjectPrefix']
        } else {
          // Create an empty project
          try {
            this.$e("a:project:create:excel");

            project = await this.$api.project.create({
              title: this.templateData.title,
              external: false
            })
            this.projectCreation = true
          } catch (e) {
            this.projectCreation = false
            this.$toast
              .error(await this._extractSdkResponseErrorMsg(e))
              .goAway(3000)
          } finally {
            clearInterval(interv)
          }

          if (!this.projectCreation) {
            // failed to create project
            return
          }

          // Create tables
          try {
            for (const t of this.localTemplateData.tables) {
              // enrich system fields if not provided
              // e.g. id, created_at, updated_at
              const systemColumns = SqlUiFactory
                .create({ client: 'sqlite3' })
                .getNewTableColumns()
                .filter(c => c.column_name != 'title')

              const table = await this.$api.dbTable.create(project.id, {
                table_name: t.table_name,
                title: '',
                columns: [...t.columns, ...systemColumns]
              })
              t.table_title = table.title
            }
            this.tableCreation = true
          } catch (e) {
            this.$toast
              .error(await this._extractSdkResponseErrorMsg(e))
              .goAway(3000)
            this.tableCreation = false
          } finally {
            clearInterval(interv)
          }
        }

        if (!this.tableCreation) {
          // failed to create table
          return
        }

        // Bulk import data
        if (this.importData) {
          this.$store.commit('loader/MutMessage', 'Importing excel data to project')
          await this.importDataToProject(this.templateData.title)
        }

        this.$router.push({
          path: `/nc/${project.id}`,
          query: {
            new: 1
          }
        })

        this.$emit('success')
      } catch (e) {
        console.log(e)
        this.$toast.error(e.message).goAway(3000)
      } finally {
        clearInterval(interv)
        this.$store.commit('loader/MutMessage', null)
        this.projectCreation = false
        this.tableCreation = false
      }
    },
    async importDataToProject(projectName) {
      let total = 0
      let progress = 0
      await Promise.all(this.localTemplateData.tables.map(v => (async(tableMeta) => {
        const tableName = tableMeta.table_title
        const data = this.importData[tableMeta.ref_table_name]
        total += data.length
        for (let i = 0; i < data.length; i += 500) {
          this.$store.commit('loader/MutMessage', `Importing data to ${projectName}: ${progress}/${total} records`)
          this.$store.commit('loader/MutProgress', Math.round(progress && 100 * progress / total))
          const batchData = this.remapColNames(data.slice(i, i + 500), tableMeta.columns)
          await this.$api.dbTableRow.bulkCreate(
            'noco',
            projectName,
            tableName,
            batchData
          )
          progress += batchData.length
        }
        this.$store.commit('loader/MutClear')
      })(v)))
    },
    remapColNames(batchData, columns) {
      return batchData.map(data => (columns || []).reduce((aggObj, col) => ({
        ...aggObj,
        [col.column_name]: data[col.ref_column_name]
      }), {})
      )
    }
  }

}
</script>

<style scoped>

</style>
