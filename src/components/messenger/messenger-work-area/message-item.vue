<template>
  <div
    class="message"
    :class="{'message--by-current-user': isCurrentUser}"
  >
    <span class="message__text">{{ message.text }}</span>
  </div>
</template>

<script lang="ts">
import {defineComponent, PropType} from "vue";
import Message from "../../../models/message.util";
import {useStore} from "vuex";

export default defineComponent({
  name: 'Message',
  props: {
    message: {
      type: Object as PropType<Message>,
      default: undefined
    }
  },
  setup(props) {
    const $store = useStore();
    const isCurrentUser = props.message?.senderLogin === $store.getters['userModule/userData'].login;

    return {
      isCurrentUser
    }
  }
})
</script>

<style lang="scss">
.message {
  border-radius: 8px;
  padding: 8px;
  background-color: #e5e5e5;

  &--by-current-user {
    background-color: #d4d8ff;
  }

  &:not(:last-child) {
    margin-bottom: 8px;
  }
}
</style>