<template>
  <div
    class="messenger overflow-hidden relative-position no-wrap flex row"
    :class="{'messenger--have-active-dialog': activeDialog}"
  >
    <dialog-list
      :dialog-list="dialogList"
      :active-dialog="activeDialog"
      @on-select-dialog="onSelectDialog"
    />
    <messenger-work-area
      v-model:message-text="messageText"
      :active-dialog="activeDialog"
      @close-dialog="deselectDialog"
      @load-messages="loadMessageList"
      @send-message="sendMessage"
    />
  </div>
</template>

<script lang="ts">
import {defineComponent, onBeforeMount, onBeforeUnmount, onMounted, ref} from 'vue';
import {socket, initSocket, killSocket} from "@/api/messenger.api";
import DialogList from '@/components/messenger/dialog-list.vue';
import {
  ESocketEvents,
  IAddMessageInNewDialogResponse,
  IAddMessageInDialogRequest,
  IAddMessageInNewDialogRequest,
  IGetMessageListRequest,
  IGetMessageListResponse
} from "@/models/socket";
import {IDialog, ISourceDialog} from "@/models/messenger";
import {IUserPublic} from "@/models/user";
import MessengerWorkArea from "@/components/messenger/messenger-work-area/messenger-work-area.vue";

export default defineComponent({
  name: 'Messenger',
  components: {
    MessengerWorkArea,
    DialogList
  },
  setup() {
    const userList = ref<IUserPublic[]>([]);
    const dialogList = ref<IDialog[]>([]);
    const activeDialog = ref<IDialog | undefined>();
    const messageText = ref<string>('');

    const onInit = (data: { userList: IUserPublic[], dialogList: ISourceDialog[] }): void => {
      userList.value = data.userList;
      dialogList.value = data.userList.map(({login, name}) => {
        const dialog = data.dialogList.find((
          {
            participantLoginList,
            isGroup
          }
        ) => participantLoginList.includes(login) && !isGroup);
        return {
          _id: dialog?._id,
          login,
          name,
          messageList: dialog?.messageList || [],
          participantLoginList: dialog?.participantLoginList || [],
          isGroup: dialog?.isGroup || false,
          isAllMessagesReceived: dialog ? dialog.isAllMessagesReceived : true,
          isLoading: false
        }
      })

      socket.on(ESocketEvents.Message, onGetMessage);
      socket.on(ESocketEvents.DialogCreate, onCreateDialog);
      socket.on(ESocketEvents.GetMessageList, onGetMessageList);
    }

    const onSelectDialog = (index: number): void => {
      deselectDialog();
      clearMessageText();
      activeDialog.value = dialogList.value[index];
    }

    const deselectDialog = () => {
      clearMessageText();
      deactivateDialogLoadingById((activeDialog.value as IDialog)?._id as string);
      activeDialog.value = undefined;
    }

    const activateDialogLoadingById = (dialogId: string): void => {
      const dialog = getDialogById(dialogId);
      if (!dialog) {
        return;
      }
      dialog.isLoading = true;
    }

    const deactivateDialogLoadingById = (dialogId: string): void => {
      const dialog = getDialogById(dialogId);
      if (!dialog) {
        return;
      }
      dialog.isLoading = false;
    }

    const loadMessageList = async (): Promise<void> => {
      if (activeDialog.value?.isAllMessagesReceived || activeDialog.value?.isLoading) {
        return;
      }
      activateDialogLoadingById((activeDialog.value as IDialog)._id as string);
      socket.emit(
        ESocketEvents.GetMessageList,
        {
          dialogId: activeDialog.value?._id,
          messageIndex: activeDialog.value?.messageList.length
        } as IGetMessageListRequest
      );
    }

    const clearMessageText = (): void => {
      messageText.value = '';
    }

    const sendMessage = async (): Promise<void> => {
      const parsedMessageText = messageText.value.replaceAll(/(^<p>)(<br>)+/gm, '<p>').replaceAll(/(<br>){3,}/g, '<br><br>');
      if (activeDialog.value?._id) {
        await sendMessageToDialog(activeDialog.value?._id as string, parsedMessageText);
        return;
      }
      await sendMessageToCompanion(activeDialog.value?.login as string, parsedMessageText);
    }

    const sendMessageToCompanion = async (companionLogin: string, text: string): Promise<void> => {
      socket.emit(ESocketEvents.AddMessageInNewDialog, {
        companionLogin,
        text
      } as IAddMessageInNewDialogRequest);
    }

    const sendMessageToDialog = async (dialogId: string, text: string): Promise<void> => {
      socket.emit(ESocketEvents.AddMessageInDialog, {
        dialogId,
        text
      } as IAddMessageInDialogRequest);
    }

    const isActiveDialog = (dialogId: string): boolean => activeDialog.value?._id === dialogId;

    const getDialogById = (dialogId: string): IDialog | undefined => {
      return dialogList.value.find((dialog) => dialog._id === dialogId)
    }

    const onGetMessage = (data: IAddMessageInNewDialogResponse): void => {
      const dialog = getDialogById(data.dialogId);
      if (!dialog) {
        return;
      }
      dialog?.messageList.push(data.message);
      isActiveDialog(dialog._id as string) && clearMessageText();
    }

    const onGetMessageList = (data: IGetMessageListResponse): void => {
      const dialog = getDialogById(data.dialogId);
      if (!dialog) {
        return;
      }
      dialog.isAllMessagesReceived = data.isAllMessagesReceived;
      dialog.messageList = [...data.messageList, ...dialog.messageList];
      setTimeout(() => {
        deactivateDialogLoadingById(data.dialogId);
      })
    }

    const onCreateDialog = (data: IDialog): void => {
      if (!data.isGroup) {
        const dialog = dialogList.value.find((dialog) => dialog.participantLoginList.includes(data.login) || data.participantLoginList.includes(dialog.login));
        if (dialog) {
          dialog._id = data._id;
          dialog.messageList = data.messageList;
          isActiveDialog(dialog._id as string) && clearMessageText();
          return;
        }
        dialogList.value.push(data);
        return;
      }
      //FIXME: group logic
    }

    onBeforeMount(() => {
      socket.on(ESocketEvents.Init, onInit);
      initSocket();
    })

    onBeforeUnmount(() => {
      socket.off(ESocketEvents.Message, onGetMessage);
      socket.off(ESocketEvents.DialogCreate, onCreateDialog);
      socket.off(ESocketEvents.GetMessageList, onGetMessageList);
      socket.off(ESocketEvents.Init, onInit);
      killSocket();
      window.removeEventListener('keydown', onKeydown);
    })

    const onKeydown = (ev: KeyboardEvent): void => {
      switch (ev.key) {
        case 'Escape':
          if (activeDialog.value) {
            deselectDialog();
            return;
          }
      }
    }

    onMounted(() => {
      window.addEventListener('keydown', onKeydown)
    })

    return {
      dialogList,
      userList,
      activeDialog,
      messageText,
      sendMessage,
      deselectDialog,
      onSelectDialog,
      loadMessageList
    }
  }
})
</script>

<style lang="scss">
@import "@/styles/default.scss";

.messenger {

  .dialog-list {
    width: 30%;

    @media (max-width: $mobile-resolution) {
      width: 100%;
    }
  }

  .messenger-work-area {
    flex-grow: 1;
    flex-shrink: 0;
    background-color: #ffffff;

    @media (max-width: $mobile-resolution) {
      position: absolute;
      height: 100%;
      width: 100%;
      z-index: 1;
      display: none;
    }
  }

  &--have-active-dialog {
    .messenger-work-area {
      display: flex;
    }
  }
}
</style>
