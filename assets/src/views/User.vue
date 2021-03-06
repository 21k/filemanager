<template>
  <div>
    <form @submit="save" class="dashboard">
      <ul id="nav">
        <li>
          <router-link to="/users">
            <i class="material-icons">keyboard_arrow_left</i> {{ $t('settings.userManagement') }}
          </router-link>
        </li>
        <li></li>
      </ul>

      <h1 v-if="id === 0">{{ $t('settings.newUser') }}</h1>
      <h1 v-else>{{ $t('settings.user') }} {{ username }}</h1>

      <p><label for="username">{{ $t('settings.username') }}</label><input type="text" v-model="username" id="username"></p>
      <p><label for="password">{{ $t('settings.password') }}</label><input type="password" :placeholder="passwordPlaceholder" v-model="password" id="password"></p>
      <p><label for="scope">{{ $t('settings.scope') }}</label><input type="text" v-model="filesystem" id="scope"></p>
      <p>
        <label for="locale">{{ $t('settings.language') }}</label>
        <languages id="locale" :selected.sync="locale"></languages>
      </p>

      <p><input type="checkbox" :disabled="admin" v-model="lockPassword"> {{ $t('settings.lockPassword') }}</p>

      <h2>{{ $t('settings.permissions') }}</h2>
      <p class="small">{{ $t('settings.permissionsHelp') }}</p>

      <p><input type="checkbox" v-model="admin"> {{ $t('settings.administrator') }}</p>
      <p><input type="checkbox" :disabled="admin" v-model="allowNew"> {{ $t('settings.allowNew') }}</p>
      <p><input type="checkbox" :disabled="admin" v-model="allowEdit"> {{ $t('settings.allowEdit') }}</p>
      <p><input type="checkbox" :disabled="admin" v-model="allowCommands"> {{ $t('settings.allowCommands') }}</p>
      <p v-show="$store.state.staticGen.length"><input type="checkbox" :disabled="admin" v-model="allowPublish"> {{ $t('settings.allowPublish') }}</p>

      <h3>{{ $t('settings.userCommands') }}</h3>
      <p class="small">{{ $t('settings.userCommandsHelp') }} <i>git svn hg</i>.</p>
      <input type="text" v-model.trim="commands">

      <h2>{{ $t('settings.rules') }}</h2>

      <p class="small">{{ $t('settings.rulesHelp1') }}</p>

      <i18n path="settings.rulesHelp2" tag="p" class="small">
        <code>allow</code><code>disallow</code><code>regex</code>
      </i18n>

      <p class="small"><strong>{{ $t('settings.examples') }}</strong></p>

      <ul class="small">
        <li><code>disallow regex \\/\\..+</code> - {{ $t('settings.ruleExample1') }}</li>
        <li><code>disallow /Caddyfile</code> - {{ $t('settings.ruleExample2') }}</li>
      </ul>

      <textarea v-model.trim="rules"></textarea>

      <h2>{{ $t('settings.customStylesheet') }}</h2>

      <textarea name="css"></textarea>

      <p>
        <button v-if="id !== 0" @click.prevent="deletePrompt" type="button" class="delete" :aria-label="$t('buttons.delete')" :title="$t('buttons.delete')">{{ $t('buttons.delete') }}</button>
        <input type="submit" :value="$t('buttons.save')">
      </p>
    </form>

    <div v-if="$store.state.show === 'deleteUser'" class="prompt">
      <h3>Delete User</h3>
      <p>Are you sure you want to delete this user?</p>
      <div>
        <button @click="deleteUser" autofocus>{{ $t('buttons.delete') }}</button>
        <button class="cancel"
          @click="closeHovers"
          :aria-label="$t('buttons.cancel')"
          :title="$t('buttons.cancel')">
          {{ $t('buttons.cancel') }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { mapMutations } from 'vuex'
import { getUser, newUser, updateUser, deleteUser } from '@/utils/api'
import Languages from '@/components/Languages'

export default {
  name: 'user',
  components: { Languages },
  data: () => {
    return {
      id: 0,
      admin: false,
      allowNew: false,
      allowEdit: false,
      allowCommands: false,
      allowPublish: false,
      lockPassword: false,
      permissions: {},
      password: '',
      username: '',
      filesystem: '',
      rules: '',
      locale: '',
      css: '',
      commands: ''
    }
  },
  computed: {
    passwordPlaceholder () {
      if (this.$route.path === '/users/new') return ''
      return this.$t('settings.avoidChanges')
    }
  },
  created () {
    this.fetchData()
  },
  watch: {
    '$route': 'fetchData',
    admin: function () {
      if (!this.admin) return
      this.allowCommands = true
      this.allowEdit = true
      this.allowNew = true
      this.allowPublish = true
      this.lockPassword = false
      for (let key in this.permissions) {
        this.permissions[key] = true
      }
    }
  },
  methods: {
    ...mapMutations(['closeHovers']),
    fetchData () {
      let user = this.$route.params[0]

      if (this.$route.path === '/users/new') {
        user = 'base'
      }

      getUser(user).then(user => {
        this.id = user.ID
        this.admin = user.admin
        this.allowCommands = user.allowCommands
        this.allowNew = user.allowNew
        this.allowEdit = user.allowEdit
        this.allowPublish = user.allowPublish
        this.lockPassword = user.lockPassword
        this.filesystem = user.filesystem
        this.username = user.username
        this.commands = user.commands.join(' ')
        this.css = user.css
        this.permissions = user.permissions
        this.locale = user.locale

        for (let rule of user.rules) {
          if (rule.allow) {
            this.rules += 'allow '
          } else {
            this.rules += 'disallow '
          }

          if (rule.regex) {
            this.rules += 'regex ' + rule.regexp.raw
          } else {
            this.rules += rule.path
          }

          this.rules += '\n'
        }

        this.rules = this.rules.trim()
      }).catch(() => {
        this.$router.push({ path: '/users/new' })
      })
    },
    capitalize (name) {
      let splitted = name.split(/(?=[A-Z])/)
      name = ''

      for (let i = 0; i < splitted.length; i++) {
        name += splitted[i].charAt(0).toUpperCase() + splitted[i].slice(1) + ' '
      }

      return name.slice(0, -1)
    },
    reset () {
      this.id = 0
      this.admin = false
      this.allowNew = false
      this.allowEdit = false
      this.allowPublish = false
      this.permissins = {}
      this.allowCommands = false
      this.lockPassword = false
      this.password = ''
      this.username = ''
      this.filesystem = ''
      this.rules = ''
      this.locale = ''
      this.css = ''
      this.commands = ''
    },
    deletePrompt (event) {
      this.$store.commit('showHover', 'deleteUser')
    },
    deleteUser (event) {
      event.preventDefault()

      deleteUser(this.id).then(location => {
        this.$router.push({ path: '/users' })
        this.$showSuccess(this.$t('settings.userDeleted'))
      }).catch(e => {
        this.$showError(e)
      })
    },
    save (event) {
      event.preventDefault()
      let user = this.parseForm()

      if (this.$route.path === '/users/new') {
        newUser(user).then(location => {
          this.$router.push({ path: location })
          this.$showSuccess(this.$t('settings.userCreated'))
        }).catch(e => {
          this.$showError(e)
        })

        return
      }

      updateUser(user).then(location => {
        if (user.ID === this.$store.state.user.ID) {
          this.$store.commit('setUser', user)
        }

        this.$showSuccess(this.$t('settings.userUpdated'))
      }).catch(e => {
        this.$showError(e)
      })
    },
    parseForm () {
      let user = {
        ID: this.id,
        username: this.username,
        password: this.password,
        lockPassword: this.lockPassword,
        filesystem: this.filesystem,
        admin: this.admin,
        allowCommands: this.allowCommands,
        allowNew: this.allowNew,
        allowEdit: this.allowEdit,
        allowPublish: this.allowPublish,
        permissions: this.permissions,
        css: this.css,
        locale: this.locale,
        commands: this.commands.split(' '),
        rules: []
      }

      let rules = this.rules.split('\n')

      for (let rawRule of rules) {
        let rule = {
          allow: true,
          path: '',
          regex: false,
          regexp: {
            raw: ''
          }
        }

        rawRule = rawRule.split(' ')

        // Skip a malformed rule
        if (rawRule.length < 2) {
          continue
        }

        // Skip a malformed rule
        if (rawRule[0] !== 'allow' && rawRule[0] !== 'disallow') {
          continue
        }

        rule.allow = (rawRule[0] === 'allow')
        rawRule.shift()

        if (rawRule[0] === 'regex') {
          rule.regex = true
          rawRule.shift()
          rule.regexp.raw = rawRule.join(' ')
        } else {
          rule.path = rawRule.join(' ')
        }

        user.rules.push(rule)
      }

      return user
    }
  }
}
</script>
