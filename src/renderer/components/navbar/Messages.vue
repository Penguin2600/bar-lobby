<template>
    <PopOutPanel :open="modelValue">
        <TabView v-model:activeIndex="activeTabIndex" class="messages-tabview">
            <TabPanel v-for="[userId, messages] of directMessages" :key="userId">
                <template #header>
                    <div class="flex-row">
                        <div>{{ getUsername(userId) }}</div>
                        <div class="flex-row close" @click="close(userId)">
                            <Icon :icon="closeThick" />
                        </div>
                    </div>
                </template>

                <div class="messages">
                    <div class="flex-col gap-sm">
                        <div
                            v-for="(message, i) in messages"
                            :key="i"
                            v-in-view.once="() => (message.read = true)"
                            :class="['message', { fromMe: message.senderUserId === myUser.userId }]"
                        >
                            {{ message.text }}
                        </div>
                    </div>
                </div>
                <div class="flex-row gap-sm flex-bottom padding-md">
                    <Textbox
                        v-model="text"
                        v-in-view="focusTextbox"
                        class="reply"
                        placeholder="Message"
                        @keyup.enter.stop="sendDirectMessage(userId, text)"
                    />
                    <Button @click="sendDirectMessage(userId, text)">Send</Button>
                </div>
            </TabPanel>

            <TabPanel>
                <template #header>
                    <Icon :icon="chatPlus" />
                </template>

                <div class="flex-col flex-grow padding-md">
                    <Textbox v-model="newMessageUserId" v-in-view="focusTextbox" class="fullwidth" label="UserID" placeholder="32452" />
                    <div class="flex-row gap-sm flex-bottom">
                        <Textbox
                            v-model="newMessage"
                            class="reply"
                            placeholder="Message"
                            @keyup.enter.stop="sendDirectMessage(newMessageUserId, newMessage)"
                        />
                        <Button @click="sendDirectMessage(newMessageUserId, newMessage)">Send</Button>
                    </div>
                </div>
            </TabPanel>
        </TabView>
    </PopOutPanel>
</template>

<script lang="ts" setup>
import { Icon } from "@iconify/vue";
import chatPlus from "@iconify-icons/mdi/chat-plus";
import closeThick from "@iconify-icons/mdi/close-thick";
import TabPanel from "primevue/tabpanel";
import { inject, Ref, ref } from "vue";

import TabView from "@/components/common/TabView.vue";
import Button from "@/components/controls/Button.vue";
import Textbox from "@/components/controls/Textbox.vue";
import PopOutPanel from "@/components/navbar/PopOutPanel.vue";
import { Message } from "@/model/messages";

const props = defineProps<{
    modelValue: boolean;
}>();

const emits = defineEmits<{
    (event: "update:modelValue", open: boolean): void;
}>();

const text = ref("");
const newMessage = ref("");
const newMessageUserId = ref("");
const directMessages = api.session.directMessages;
const myUser = api.session.onlineUser;
const activeTabIndex = ref(Math.max(directMessages.size - 1, 0));

const toggleMessages = inject<Ref<(open?: boolean, userId?: number) => void>>("toggleMessages")!;
const toggleFriends = inject<Ref<(open?: boolean) => void>>("toggleFriends")!;
const toggleDownloads = inject<Ref<(open?: boolean) => void>>("toggleDownloads")!;

toggleMessages.value = async (open?: boolean, userIdToActivate?: number) => {
    if (open) {
        toggleFriends.value(false);
        toggleDownloads.value(false);
    }

    emits("update:modelValue", open ?? !props.modelValue);

    if (userIdToActivate) {
        let i = 0;
        for (const [userId] of directMessages) {
            if (userId === userIdToActivate) {
                break;
            }
            i++;
        }
        activeTabIndex.value = i;
    }
};

function focusTextbox(el: HTMLElement) {
    if (el.firstElementChild && el.firstElementChild instanceof HTMLElement) {
        el.firstElementChild.focus();
    }
}

function getUsername(userId: number) {
    return api.session.getUserById(userId)?.username ?? "??";
}

async function sendDirectMessage(userIdInput: number | string, messageText: string) {
    const userId = typeof userIdInput === "string" ? parseInt(userIdInput) : userIdInput;

    newMessageUserId.value = "";
    newMessage.value = "";
    text.value = "";

    const response = await api.comms.request("c.communication.send_direct_message", {
        recipient_id: userId,
        message: messageText,
    });

    if (response.result === "success") {
        const chatlog = api.session.directMessages.get(userId);
        const message: Message = {
            senderUserId: api.session.onlineUser.userId,
            text: messageText,
            type: "direct-message",
            read: true,
        };
        if (chatlog) {
            chatlog.push(message);
        } else {
            api.session.directMessages.set(userId, [message]);
        }
    }
}

function close(userId: number) {
    api.session.directMessages.delete(userId);
}
</script>

<style lang="scss" scoped>
.messages-tabview,
:deep(.p-tabview-panels),
:deep(.p-tabview-panel) {
    display: flex;
    flex-direction: column;
    flex-grow: 1;
    padding: 0 !important;
}
:deep(.p-tabview-header-action) {
    overflow: unset;
}
.messages {
    display: flex;
    flex-direction: column-reverse;
    overflow-y: scroll;
    padding: 10px;
    flex: 1 1 auto;
    height: 0;
}
.message {
    word-break: break-word;
    padding: 4px 8px;
    user-select: text;
    display: flex;
    flex-direction: row;
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 3px;
    align-self: flex-start;
    &.fromMe {
        align-self: flex-end;
        background: rgba(240, 240, 240, 0.247);
    }
}
.reply-container {
    padding: 10px;
    padding-right: 20px;
}
.reply {
    width: 100%;
}
.close {
    position: absolute;
    height: 100%;
    top: 0;
    right: 0;
    display: flex;
    align-items: center;
    background: none;
    border: none;
    opacity: 0.5;
    padding: 10px;
    &:hover {
        opacity: 1;
    }
}
</style>
