<template>
  <Modal ref="modal" @close="cleanup" v-if="show">
    <template #title>
      {{ t('memories', 'Rename person') }}
    </template>

    <div class="fields">
      <NcTextField
        class="field"
        :autofocus="true"
        :value.sync="input"
        :label="t('memories', 'Name')"
        :label-visible="false"
        :placeholder="t('memories', 'Name')"
        @keypress.enter="save()"
      />
    </div>

    <template #buttons>
      <NcButton class="button" type="primary" :disabled="!canSave" @click="save">
        {{ t('memories', 'Update') }}
      </NcButton>
    </template>
  </Modal>
</template>

<script lang="ts">
import { defineComponent } from 'vue';

import { showError } from '@nextcloud/dialogs';

import NcButton from '@nextcloud/vue/dist/Components/NcButton';
const NcTextField = () => import('@nextcloud/vue/dist/Components/NcTextField');

import Modal from './Modal.vue';
import ModalMixin from './ModalMixin';

import * as utils from '@services/utils';
import * as dav from '@services/dav';

export default defineComponent({
  name: 'FaceEditModal',
  components: {
    NcButton,
    NcTextField,
    Modal,
  },

  mixins: [ModalMixin],

  emits: [],

  data: () => ({
    input: String(),
  }),

  computed: {
    name() {
      return this.$route.params.name;
    },

    user() {
      return this.$route.params.user;
    },

    canSave() {
      return this.input && this.name !== this.input && isNaN(Number(this.input));
    },
  },

  methods: {
    open() {
      if (this.user !== utils.uid) {
        showError(this.t('memories', 'Only user "{user}" can update this person', { user: this.user }));
        return;
      }

      this.input = isNaN(Number(this.name)) ? this.name : String();
      this.show = true;
    },

    cleanup() {
      this.show = false;
    },

    async save() {
      if (!this.canSave) return;

      try {
        if (this.routeIsRecognize) {
          await dav.recognizeRenameFace(this.user, this.name, this.input);
        } else {
          await dav.faceRecognitionRenamePerson(this.name, this.input);
        }

        this.$router.replace({
          name: this.$route.name as string,
          params: { user: this.user, name: this.input },
        });

        this.close();
      } catch (error) {
        console.log(error);
        showError(
          this.t('memories', 'Failed to rename {oldName} to {name}.', {
            oldName: this.name,
            name: this.input,
          }),
        );
      }
    },
  },
});
</script>

<style lang="scss" scoped>
.fields {
  margin-top: 8px;
}
</style>
