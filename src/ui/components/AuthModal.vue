<template>
  <div v-show="false">
    <div
      ref="root"
      class="app-modal app-bg"
    >

      <ion-card-header
        color="light"
        class="app-modal-header"
      >
        <ion-card-subtitle style="font-size: 16px;">{{ modalTitles[stage] }}</ion-card-subtitle>
      </ion-card-header>

      <ion-list
        v-if="stage === 0"
        class="app-modal-content"
        lines="full"
      >
        <auth-method-item
          v-for="p of authMethods"
          :provider="p"
          :key="p.type"
          @continue="continueCb"
        ></auth-method-item>
      </ion-list>

      <ion-spinner
        v-if="stage === 1"
        color="primary"
        style="align-self: center; flex: 1;"
      />

      <user-profile-box
        v-if="stage === 2"
        :publicKey="pubKey"
      />

      <ion-buttons
        v-if="stage === 0 || stage === 2"
        class="app-modal-buttons"
      >
        <ion-button
          color="danger"
          @click="cancelCb"
        >Cancel</ion-button>
        <ion-button
          v-if="stage === 2"
          color="primary"
          @click="continueCb"
        >Continue</ion-button>
      </ion-buttons>

    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, createApp } from 'vue'
import { IonCardHeader, IonCardSubtitle, IonList, IonSpinner, IonButtons, IonButton } from '@ionic/vue'
import { Identity, UserPubKeyType } from '@/identity'
import { listProviders, IdentityProvider } from '@/identity/provider'
import type { IPFS } from 'ipfs'

import AuthMethodItem from './AuthMethodItem.vue'
import UserProfileBox from './UserProfileBox.vue'
import { showModal } from '../mixins/modal'

type AuthMethod = IdentityProvider

export const ERR_USER_ABORTED = new Error('user aborted')

const AuthModal = defineComponent({
  components: {
    IonCardHeader,
    IonCardSubtitle,
    IonList,
    IonSpinner,
    IonButtons,
    IonButton,
    AuthMethodItem,
    UserProfileBox,
  },
  data () {
    return {
      modalTitle: '',
      authMethods: [] as AuthMethod[],
      stage: 0,
      pubKey: undefined as UserPubKeyType | undefined,
      continueCb: (() => undefined) as (provider: IdentityProvider, inputs: Record<string, any>) => void,
      cancelCb: (): void => undefined,
      modalTitles: [
        'Select an Authentication Method',
        'Authorizing...',
        'Confirmation. Is that you?',
      ],
    }
  },
  methods: {
    showModal (): Promise<HTMLIonModalElement> {
      return showModal(this.$refs.root as any, 'app-modal-root')
    },
    async requestIdentity (): Promise<Identity> {
      this.init()
      const modal = await this.showModal()

      try {
        // wait for user response
        const [provider, inputs]: [IdentityProvider, Record<string, any>] = await new Promise((resolve, reject) => {
          this.continueCb = (...args): void => {
            resolve(args)
          }
          this.cancelCb = (): void => {
            reject(ERR_USER_ABORTED)
          }
        })
        this.stage++

        console.info('inputs', { ...inputs })

        // authorizing
        const [identity] = await provider.requestIdentities(inputs as any)

        // load user profile
        const pubKey = await identity.publicKey()
        this.pubKey = pubKey
        this.stage++

        // wait for user response
        await new Promise((resolve, reject) => {
          this.continueCb = resolve
          this.cancelCb = (): void => {
            reject(ERR_USER_ABORTED)
            // todo: should start over
          }
        })

        return identity
      } finally {
        // close
        await modal.dismiss()
      }
    },
    init (): void {
      this.stage = 0
      this.authMethods = listProviders(false)
    },
  },
})

export default AuthModal

/**
 * @param ipfs - the `ipfs` injection
 */
export const requestIdentity = async (ipfs: IPFS): Promise<Identity> => {
  // create a void HTML element, and trick Vue 
  // to instantiate the `AuthModal` component
  const voidEl = document.createElement('div')
  const c = createApp(AuthModal)
  c.provide('ipfs', ipfs)
  const vm = c.mount(voidEl) as InstanceType<typeof AuthModal>

  const identity = await vm.requestIdentity()

  c.unmount(voidEl) // GC

  return identity
}
</script>

<style>
  .app-modal-root {
    --border-radius: 4px;
    --max-width: 25.5rem;
    --box-shadow: 0 28px 48px rgba(0, 0, 0, 0.4);
    padding: 1em;
  }

  body.dark .app-modal-root {
    --box-shadow: 0 28px 48px rgba(255, 255, 255, 0.4);
  }
</style>

<style scoped>
  .app-modal {
    justify-content: flex-start;
  }

  .app-modal-header {
    border-bottom: var(--app-border);
  }

  .app-modal-content {
    padding: 0;
    overflow-y: auto;
  }

  .app-modal-buttons {
    margin-top: auto;
    border-top: var(--app-border);
    justify-content: space-between;
  }

  .app-modal-buttons > ion-button {
    margin: 0;
  }
</style>
