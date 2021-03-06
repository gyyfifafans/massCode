<template>
  <AppForm>
    <AppFormItem label="Storage">
      <div class="preferences__form-item">
        <AppInput
          v-model="storagePath"
          :readonly="true"
          bordered
          size="small"
          class="preferences__input"
        />
        <AppButton @click="onChangeStorage">
          Move storage
        </AppButton>
        <AppButton @click="onOpenStorage">
          Open storage
        </AppButton>
      </div>
      <div class="desc">
        To use sync services like iCloud Drive, Google Drive of Dropbox, simply
        move storage to the corresponding synced folders.
      </div>
    </AppFormItem>
    <AppFormItem label="Backup">
      <div class="preferences__form-item">
        <AppInput
          v-model="app.backupPath"
          :readonly="true"
          bordered
          size="small"
          class="preferences__input"
        />
        <AppButton @click="onMoveBackup">
          Change folder
        </AppButton>
        <AppButton @click="onBackup">
          Backup now
        </AppButton>
      </div>
      <div class="desc">
        Backup will be created automatically when massCode is running.
      </div>
      <h5>Backups</h5>
      <div
        ref="backup"
        class="backups"
      >
        <div
          v-for="(i, index) in backups"
          :key="index"
          class="backups__item"
        >
          <!-- <span class="backups__item-count">210 snippets</span> -->
          <span class="backups__item-date">{{ i.label }}</span>
          <span
            class="backups__item-action"
            @click="onRestore(i.date)"
          >Restore</span>
        </div>
      </div>
    </AppFormItem>
    <AppFormItem label="Count">
      {{ countText }}
    </AppFormItem>
  </AppForm>
</template>

<script>
import { mapState, mapGetters } from 'vuex'
import { dialog } from '@@/lib'
import { ipcRenderer } from 'electron'
import db from '@/datastore'
import PerfectScrollbar from 'perfect-scrollbar'

export default {
  name: 'Storage',

  data () {
    return {
      ps: null
    }
  },

  computed: {
    ...mapState(['app']),
    ...mapGetters('app', ['backups']),
    ...mapGetters('snippets', ['count']),
    storagePath: {
      get () {
        return this.app.storagePath
      },
      set (v) {
        this.$store.commit('app/SET_STORAGE_PATH', v)
      }
    },
    countText () {
      return this.count === 1
        ? `${this.count} snippet`
        : `${this.count} snippets`
    }
  },

  watch: {
    $route (route) {
      if (route.name === 'preferences') {
        this.getData()
      }
    }
  },

  created () {
    this.getData()
  },

  mounted () {
    this.initPS()
  },

  methods: {
    async onChangeStorage () {
      const dir = dialog.showOpenDialogSync({
        properties: ['openDirectory', 'createDirectory']
      })
      if (dir) {
        const path = dir[0]
        try {
          await db.move(path)
          this.updateData()
          this.$store.commit('app/SET_STORAGE_PATH', path)
        } catch (err) {
          ipcRenderer.send('message', {
            message: 'Error',
            type: 'error',
            detail:
              'Folder already contains db files. Please select another folder.'
          })
        }
      }
    },
    async onOpenStorage () {
      const dir = dialog.showOpenDialogSync({
        properties: ['openDirectory']
      })
      if (dir) {
        const path = dir[0]
        db.import(path)
        this.updateData()
        this.$store.commit('app/SET_STORAGE_PATH', path)
      }
    },
    async updateData () {
      this.$store.dispatch('snippets/setSelected', null)
      this.$store.dispatch('folders/setSelectedFolder', 'allSnippets')
      await this.$store.dispatch('folders/getFolders')
      await this.$store.dispatch('snippets/getSnippets')
    },
    async onBackup () {
      await db.removeEarliestBackup()
      await db.backup()
      this.$store.dispatch('app/getBackups')
      this.$nextTick(() => {
        this.ps.update()
      })
    },
    async onRestore (time) {
      const buttonId = dialog.showMessageBoxSync({
        message: 'Are you sure you want to restore for this timestamp?',
        detail:
          'During restore from backup, the current library will be overwritten.',
        buttons: ['Confirm', 'Cancel'],
        defaultId: 0,
        cancelId: 1
      })

      if (buttonId === 1) return

      await db.restoreFromBackup(time)
      this.updateData()
    },
    async onMoveBackup () {
      const dir = dialog.showOpenDialogSync({
        properties: ['openDirectory', 'createDirectory']
      })
      if (dir) {
        const path = dir[0]
        try {
          db.moveBackup(path)
          this.$store.commit('app/SET_BACKUP_PATH', path)
        } catch (err) {
          ipcRenderer.send('message', {
            message: 'Error',
            type: 'error',
            detail: 'asdsada'
          })
        }
      }
    },
    getData () {
      this.$store.dispatch('app/getBackups')
      this.$store.dispatch('snippets/getSnippetsCount')
    },
    initPS () {
      this.ps = new PerfectScrollbar(this.$refs.backup)
    }
  }
}
</script>

<style lang="scss">
.backups {
  // margin-top: var(--spacing-sm);
  border: 1px solid var(--color-border);
  max-width: 300px;
  height: 120px;
  padding: 2px var(--spacing-xs);
  position: relative;
  &__item {
    padding: 4px 0;
    display: flex;
    justify-content: space-between;
    -webkit-user-select: none;
    font-size: var(--text-sm);
    span {
      + span {
        margin-left: var(--spacing-xs);
      }
    }
    &-count {
      flex-grow: 1;
    }
    &-action {
      color: var(--color-contrast-medium);
      cursor: default;
      &:hover {
        color: var(--color-text);
      }
    }
  }
}
</style>
