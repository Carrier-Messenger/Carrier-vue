<template>
  <div id="chat" v-if="chat" ref="chat">
    <div id="messages" ref="messages">
      <h1 class="chatTitle">Welcome in {{ chat.name }}</h1>
      <button
        id="load-more"
        @click="loadMoreMessages"
        v-if="!loadedAll"
        class="linkBlack"
      >
        Load more messages
      </button>
      <p v-else>Looks like there are no more messages</p>
      <Message
        v-for="(message, index) in messages"
        :key="index"
        :message="message"
        :fromWS="false"
      />
      <Message
        v-for="(message, index) in WSMessages"
        :key="index"
        :message="message"
        :fromWS="true"
      />
      <div id="input">
        <input type="text" v-model="message" @keyup.enter="sendMessage" />
        <input
          title=" "
          value=""
          class="sendFile"
          type="file"
          id="images"
          accept="image/png, image/jpg, image/jpeg"
          @change="addFile"
          ref="fileupload"
          multiple
        />
        <button @click="sendMessage">Send</button>
      </div>
    </div>
    <div id="user-section">
      <UserList
        :chat="chat"
        @addAdmin="addAdmin($event)"
        @removeAdmin="removeAdmin($event)"
        @removeUser="removeUser($event)"
      />
      <div style="margin-top: 50px"></div>
      <button @click="leave" class="linkBlack">Leave chat</button>
      <button @click="deleteChat" v-if="chat.is_admin" class="linkBlack">
        Delete chat
      </button>
    </div>
  </div>
</template>

<script>
import Chat from "@/scripts/chat";
import Message from "@/components/Message";
import { WSDOMAIN, LIMIT } from "@/settings";
import JWT from "@/scripts/jwt";
import UserList from "@/components/UserList";

export default {
  name: "ChatView",
  components: {
    Message,
    UserList,
  },
  data() {
    return {
      chat: null,
      messages: [],
      WSMessages: [],

      message: "",
      images: [],
      base64: [],

      offset: 0,
      limit: LIMIT,
      lastMessage: 0,

      loadedAll: false,

      socket: WebSocket,
    };
  },
  methods: {
    onAllEneded() {
      if (this.base64.length === this.images.length) {
        this.socket.send(
          JSON.stringify({ message: this.message, images: this.base64 })
        );
        this.$refs.fileupload.value = null;
        this.base64 = [];
      }
    },
    sendMessage() {
      if (this.images.length <= 0) {
        this.socket.send(JSON.stringify({ message: this.message }));
        return;
      }
      for (let image of this.images) {
        const reader = new FileReader();
        let rawImg;
        reader.onloadend = () => {
          rawImg = reader.result;
          this.base64.push(rawImg);
          this.onAllEneded();
        };
        reader.readAsDataURL(image);
      }
    },
    addFile(event) {
      const allowedExtensions = /(\.jpg|\.jpeg|\.png)$/i;
      if (!allowedExtensions.exec(event.target.files[0].name)) {
        this.$refs.fileupload.value = null;
        return;
      }
      this.images = event.target.files;
      this.message = "";
    },
    async leave() {
      Chat.leave(this.$route.params.id);
      this.$router.push({ name: "ChatView" });
      this.$emit("removechat", this.$route.params.id);
    },
    async deleteChat() {
      Chat.delete(this.$route.params.id);
      this.$router.push({ name: "ChatView" });
      this.$emit("removechat", this.$route.params.id);
    },
    async loadMoreMessages() {
      const newMessages = (
        await Chat.getChatMessages(
          this.$route.params.id,
          this.offset,
          this.limit,
          this.lastMessage
        )
      ).reverse();

      if (newMessages.length < this.limit) {
        this.loadedAll = true;
      }

      this.messages = newMessages.concat(this.messages);
      this.offset += this.limit;
    },
    removeAdmin(id) {
      const user = this.chat.users.find((user) => user.id === id);
      this.chat.users.find((user) => user.id === id).is_admin = false;
      this.chat.creators.find((user) => user.id === id).is_admin = false;
      this.chat.creators.splice(
        this.chat.creators.indexOf(user),
        1
      );
    },
    addAdmin(id) {
      const user = this.chat.users.find((user) => user.id === id);
      this.chat.creators.push(user);
      this.chat.users.find((user) => user.id === id).is_admin = true;
    },
    removeUser(id) {
      const user = this.chat.users.find((user) => user.id === id);

      this.chat.users.splice(this.chat.users.indexOf(user), 1);

      if (this.chat.creators.includes(user)) {
        this.chat.creators.splice(
          this.chat.creators.indexOf(user),
          1
        );
      }
    },
  },

  async mounted() {
    this.chat = await Chat.getChatInfo(this.$route.params.id);

    this.socket = new WebSocket(
      `${WSDOMAIN}ws/chat/${
        this.$route.params.id
      }/?token=${await JWT.getToken()}`
    );
    this.socket.onmessage = (e) => {
      let message = JSON.parse(e.data);
      message.created_at = new Date(message.created_at);
      this.WSMessages.push(message);
      this.$emit("changelastmessage", {
        id: this.chat.id,
        message: message,
      });
    };
    this.socket.onclose = () => {
      alert("You have problems with internet connection.");
      this.$router.push({ name: "Self" });
    };

    this.messages = (
      await Chat.getChatMessagesFirst(this.$route.params.id, 0, this.limit)
    ).reverse();

    this.messages.forEach((message) => {
      message.created_at = new Date(message.created_at);
    });

    if (this.messages.length < this.limit) {
      this.loadedAll = true;
      return;
    }

    this.lastMessage = this.messages[this.messages.length - 1].id;
    this.offset += this.limit;
    //this.$refs.messages.scrollTop = this.$refs.messages.scrollHeight;
  },
};
</script>

<style scoped>
</style>