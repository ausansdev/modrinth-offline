<template>
  <div
    v-if="mode !== 'isolated'"
    ref="button"
    v-tooltip.right="'Minecraft accounts'"
    class="button-base avatar-button"
    :class="{ expanded: mode === 'expanded' }"
    @click="showCard = !showCard"
  >
    <Avatar
      :size="mode === 'expanded' ? 'xs' : 'sm'"
      :src="
        selectedAccount
          ? `https://mc-heads.net/avatar/${selectedAccount.id}/128`
          : 'https://launcher-files.modrinth.com/assets/steve_head.png'
      "
    />
  </div>

  <Modal ref="chooseAccountTypeModal" header="Choose new account type">
    <div class="choose-account-modal">
      <div class="account-types">
        <Button @click="login()">
          <SSOMicrosoftIcon />Microsoft
        </Button>
        <Button @click="$refs.chooseAccountTypeModal.hide(); $refs.chooseOfflineUsernameModal.show()">
          <ClientIcon />Offline
        </Button>
      </div>
    </div>
  </Modal>

  <Modal ref="chooseOfflineUsernameModal" header="Choose username for your offline account">
    <div class="choose-offline-username-modal">
      <input type="text" placeholder="Username" v-model="offlineUsername" />
      <Button icon-only color="primary" @click="offlineLogin(offlineUsername)">
        <LogInIcon />
      </Button>
    </div>
  </Modal>

  <transition name="fade">
    <Card
      v-if="showCard || mode === 'isolated'"
      ref="card"
      class="account-card"
      :class="{ expanded: mode === 'expanded', isolated: mode === 'isolated' }"
    >
      <div v-if="selectedAccount" class="selected account">
        <Avatar size="xs" :src="`https://mc-heads.net/avatar/${selectedAccount.id}/128`" />
        <div>
          <h4>{{ selectedAccount.username }}</h4>
          <p>Selected</p>
        </div>
        <Button v-tooltip="'Log out'" icon-only color="raised" @click="logout(selectedAccount.id)">
          <TrashIcon />
        </Button>
      </div>
      <div v-else class="logged-out account">
        <h4>Not signed in</h4>
        <Button v-tooltip="'Log in'" icon-only color="primary"
            @click="$refs.chooseAccountTypeModal.show()">
          <LogInIcon />
        </Button>
      </div>
      <div v-if="displayAccounts.length > 0" class="account-group">
        <div v-for="account in displayAccounts" :key="account.id" class="account-row">
          <Button class="option account" @click="setAccount(account)">
            <Avatar :src="`https://mc-heads.net/avatar/${account.id}/128`" class="icon" />
            <p>{{ account.username }}</p>
          </Button>
          <Button v-tooltip="'Log out'" icon-only @click="logout(account.id)">
            <TrashIcon />
          </Button>
        </div>
      </div>
      <Button v-if="accounts.length > 0" @click="$refs.chooseAccountTypeModal.show()">
        <PlusIcon />
        Add account
      </Button>
    </Card>
  </transition>
</template>

<script setup>
import { PlusIcon, TrashIcon, LogInIcon, SSOMicrosoftIcon, ClientIcon } from '@modrinth/assets'
import { Avatar, Button, Card, Modal } from '@modrinth/ui'
import { ref, computed, onMounted, onBeforeUnmount, onUnmounted } from 'vue'
import {
  users,
  remove_user,
  set_default_user,
  login as login_flow,
  offline_login,
  get_default_user,
} from '@/helpers/auth'
import { handleError } from '@/store/state.js'
import { mixpanel_track } from '@/helpers/mixpanel'
import { process_listener } from '@/helpers/events'
import { handleSevereError } from '@/store/error.js'

defineProps({
  mode: {
    type: String,
    required: true,
    default: 'normal',
  },
})

const emit = defineEmits(['change'])

const chooseAccountTypeModal = ref(null)
const chooseOfflineUsernameModal = ref(null)

const accounts = ref({})
const defaultUser = ref()
const offlineUsername = ref('')

async function refreshValues() {
  defaultUser.value = await get_default_user().catch(handleError)
  accounts.value = await users().catch(handleError)
}
defineExpose({
  refreshValues,
})
await refreshValues()

const displayAccounts = computed(() =>
  accounts.value.filter((account) => defaultUser.value !== account.id),
)

const selectedAccount = computed(() =>
  accounts.value.find((account) => account.id === defaultUser.value),
)

async function setAccount(account) {
  defaultUser.value = account.id
  await set_default_user(account.id).catch(handleError)
  emit('change')
}

async function login() {
  const loggedIn = await login_flow().catch(handleSevereError)

  if (loggedIn) {
    await setAccount(loggedIn)
    await refreshValues()
  }

  mixpanel_track('AccountLogIn')
}

async function offlineLogin(username) {
  if (username.length == 0) {
    return
  }

  chooseOfflineUsernameModal.value.hide()
  const loggedIn = await offline_login(username).catch(handleSevereError)

  if (loggedIn) {
    await setAccount(loggedIn)
    await refreshValues()
  }

  mixpanel_track('AccountLogIn')
}

const logout = async (id) => {
  await remove_user(id).catch(handleError)
  await refreshValues()
  if (!selectedAccount.value && accounts.value.length > 0) {
    await setAccount(accounts.value[0])
    await refreshValues()
  } else {
    emit('change')
  }
  mixpanel_track('AccountLogOut')
}

let showCard = ref(false)
let card = ref(null)
let button = ref(null)
const handleClickOutside = (event) => {
  const elements = document.elementsFromPoint(event.clientX, event.clientY)
  if (
    card.value &&
    card.value.$el !== event.target &&
    !elements.includes(card.value.$el) &&
    !button.value.contains(event.target)
  ) {
    showCard.value = false
  }
}

const unlisten = await process_listener(async (e) => {
  if (e.event === 'launched') {
    await refreshValues()
  }
})

onMounted(() => {
  window.addEventListener('click', handleClickOutside)
})

onBeforeUnmount(() => {
  window.removeEventListener('click', handleClickOutside)
})

onUnmounted(() => {
  unlisten()
})
</script>

<style scoped lang="scss">
.selected {
  background: var(--color-brand-highlight);
  border-radius: var(--radius-lg);
  color: var(--color-contrast);
  gap: 1rem;
}

.logged-out {
  background: var(--color-bg);
  border-radius: var(--radius-lg);
  gap: 1rem;
}

.account {
  width: max-content;
  display: flex;
  align-items: center;
  text-align: left;
  padding: 0.5rem 1rem;

  h4,
  p {
    margin: 0;
  }
}

.account-card {
  position: absolute;
  display: flex;
  flex-direction: column;
  top: 0.5rem;
  left: 5.5rem;
  z-index: 11;
  gap: 0.5rem;
  padding: 1rem;
  border: 1px solid var(--color-button-bg);
  width: max-content;
  user-select: none;
  -ms-user-select: none;
  -webkit-user-select: none;
  max-height: 98vh;
  overflow-y: auto;

  &::-webkit-scrollbar-track {
    border-top-right-radius: 1rem;
    border-bottom-right-radius: 1rem;
  }

  &::-webkit-scrollbar {
    border-top-right-radius: 1rem;
    border-bottom-right-radius: 1rem;
  }

  &.hidden {
    display: none;
  }

  &.expanded {
    left: 13.5rem;
  }

  &.isolated {
    position: relative;
    left: 0;
    top: 0;
  }
}

.accounts-title {
  font-size: 1.2rem;
  font-weight: bolder;
}

.account-group {
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.option {
  width: calc(100% - 2.25rem);
  background: var(--color-raised-bg);
  color: var(--color-base);
  box-shadow: none;

  img {
    margin-right: 0.5rem;
  }
}

.icon {
  --size: 1.5rem !important;
}

.account-row {
  display: flex;
  flex-direction: row;
  gap: 0.5rem;
  vertical-align: center;
  justify-content: space-between;
  padding-right: 1rem;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

.avatar-button {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  color: var(--color-base);
  background-color: var(--color-raised-bg);
  border-radius: var(--radius-md);
  width: 100%;
  text-align: left;

  &.expanded {
    border: 1px solid var(--color-button-bg);
    padding: 1rem;
  }
}

.avatar-text {
  margin: auto 0 auto 0.25rem;
  display: flex;
  flex-direction: column;
}

.text {
  width: 6rem;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.accounts-text {
  display: flex;
  align-items: center;
  gap: 0.25rem;
  margin: 0;
}

.qr-code {
  background-color: white !important;
  border-radius: var(--radius-md);
}

.modal-body {
  display: flex;
  flex-direction: row;
  gap: var(--gap-lg);
  align-items: center;
  padding: var(--gap-xl);

  .modal-text {
    display: flex;
    flex-direction: column;
    gap: var(--gap-sm);
    width: 100%;

    h2,
    p {
      margin: 0;
    }

    .code-text {
      display: flex;
      flex-direction: row;
      gap: var(--gap-xs);
      align-items: center;

      .code {
        background-color: var(--color-bg);
        border-radius: var(--radius-md);
        border: solid 1px var(--color-button-bg);
        font-family: var(--mono-font);
        letter-spacing: var(--gap-md);
        color: var(--color-contrast);
        font-size: 2rem;
        font-weight: bold;
        padding: var(--gap-sm) 0 var(--gap-sm) var(--gap-md);
      }

      .btn {
        width: 2.5rem;
        height: 2.5rem;
      }
    }
  }
}

.button-row {
  display: flex;
  flex-direction: row;
}

.modal {
  position: absolute;
}

.code {
  color: var(--color-brand);
  padding: 0.05rem 0.1rem;
  // row not column
  display: flex;

  .card {
    background: var(--color-base);
    color: var(--color-contrast);
    padding: 0.5rem 1rem;
  }
}

.choose-account-modal {
  display: flex;
  flex-direction: column;
  padding: 2rem 0.1rem;
  justify-content: center;
  align-items: center;
  width: 100%;
  gap: 1.5rem;
}

.account-types {
  display: flex;
  gap: 1rem;
}

.choose-offline-username-modal {
  display: flex;
  padding: 2rem;
  justify-content: center;
  align-items: center;
  gap: 0.5rem;
}
</style>
