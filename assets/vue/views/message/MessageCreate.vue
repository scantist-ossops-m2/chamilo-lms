<template>
  <!--        :handle-submit="onSendMessageForm"-->
  <MessageForm
    v-model:attachments="attachments"
    :values="item"
  >
    <!--          @input="v$.item.receiversTo.$touch()"-->

    <div
      v-if="sendToUser"
      class="field"
    >
      <span v-t="'To'" />
      <BaseUserAvatar :image-url="sendToUser.illustrationUrl" />
      <span v-text="sendToUser.fullName" />
    </div>
    <BaseAutocomplete
      v-else
      id="to"
      v-model="usersTo"
      :label="t('To')"
      :search="asyncFind"
      is-multiple
    />

    <BaseAutocomplete
      v-if="!sendToUser"
      id="cc"
      v-model="usersCc"
      :label="t('Cc')"
      :search="asyncFind"
      is-multiple
    />

    <div class="field">
      <BaseTinyEditor
        v-model="item.content"
        editor-id="message"
        required
      />
    </div>

    <BaseButton
      :label="t('Send')"
      icon="plus"
      type="primary"
      @click="onSubmit"
    />
  </MessageForm>
  <Loading :visible="isLoading || isLoadingUser" />
</template>

<script setup>
import { useStore } from "vuex"
import MessageForm from "../../components/message/Form.vue"
import Loading from "../../components/Loading.vue"
import { computed, ref } from "vue"
import BaseAutocomplete from "../../components/basecomponents/BaseAutocomplete.vue"
import BaseButton from "../../components/basecomponents/BaseButton.vue"
import { useI18n } from "vue-i18n"
import { useRoute, useRouter } from "vue-router"
import { MESSAGE_TYPE_INBOX } from "../../components/message/constants"
import userService from "../../services/userService"
import BaseUserAvatar from "../../components/basecomponents/BaseUserAvatar.vue"
import { useNotification } from "../../composables/notification"
import { capitalize } from "lodash"
import BaseTinyEditor from "../../components/basecomponents/BaseTinyEditor.vue"
import { useSecurityStore } from "../../store/securityStore"

const store = useStore()
const securityStore = useSecurityStore()
const router = useRouter()
const route = useRoute()
const { t } = useI18n()

const notification = useNotification()

const asyncFind = async (query) => {
  const { items } = await userService.findByUsername(query)

  return items.map((member) => ({
    name: member.fullName,
    value: member["@id"],
  }))
}

const item = ref({
  sender: securityStore.user["@id"],
  receivers: [],
  msgType: MESSAGE_TYPE_INBOX,
  title: "",
  content: "",
})

const attachments = ref([])

const usersTo = ref([])

const usersCc = ref([])

const receiversTo = computed(() =>
  usersTo.value.map((userTo) => ({
    receiver: userTo.value,
    receiverType: 1,
  })),
)

const receiversCc = computed(() =>
  usersCc.value.map((userCc) => ({
    receiver: userCc.value,
    receiverType: 2,
  })),
)

const isLoading = computed(() => store.getters["message/isLoading"])
const messageCreated = computed(() => store.state.message.created)

const onSubmit = () => {
  item.value.receivers = [...receiversTo.value, ...receiversCc.value]

  store.dispatch("message/create", item.value).then(() => {
    if (attachments.value.length > 0) {
      attachments.value.forEach((attachment) =>
        store.dispatch("messageattachment/createWithFormData", {
          messageId: messageCreated.value.id,
          file: attachment,
        }),
      )
    }

    router.push({
      name: "MessageList",
      query: route.query,
    })
  })
}

const browser = (callback, value, meta) => {
  let url = "/resources/personal_files/"

  if (meta.filetype === "image") {
    url = url + "&type=images"
  } else {
    url = url + "&type=files"
  }

  window.addEventListener("message", function (event) {
    var data = event.data
    if (data.url) {
      url = data.url
      console.log(meta) // {filetype: "image", fieldname: "src"}
      callback(url)
    }
  })

  tinymce.activeEditor.windowManager.openUrl(
    {
      url: url,
      title: "file manager",
    },
    {
      oninsert: function (file, fm) {
        var url, reg, info

        url = fm.convAbsUrl(file.url)

        info = file.name + " (" + fm.formatSize(file.size) + ")"

        if (meta.filetype === "file") {
          callback(url, { text: info, title: info })
        }

        if (meta.filetype === "image") {
          callback(url, { alt: info })
        }

        if (meta.filetype === "media") {
          callback(url)
        }
      },
    },
  )
}

const isLoadingUser = ref(false)
const sendToUser = ref()

if (route.query.send_to_user) {
  isLoadingUser.value = true

  userService
    .find("/api/users/" + parseInt(route.query.send_to_user))
    .then((user) => {
      sendToUser.value = user

      usersTo.value.push({
        name: user.fullName,
        value: user["@id"],
      })

      if (route.query.prefill) {
        const prefill = capitalize(route.query.prefill)

        item.value.title = t(prefill + "Title")
        item.value.content = t(prefill + "Content", [
          user.firstname,
          securityStore.user.firstname,
          securityStore.user.firstname,
        ])
      }
    })
    .catch((e) => notification.showErrorNotification(e))
    .finally(() => (isLoadingUser.value = false))
}
</script>
