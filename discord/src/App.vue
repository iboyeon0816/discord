<template>
	<div v-if="showApp" id="app">
    사용자 이름 <input v-model="username" type="text">
    <button @click="connect">Connect</button>
    <div v-if="connectionErrorMessage" class="error">{{ connectionErrorMessage }}</div>
	</div>
  <div v-else>
    <h3>사용자 이름: {{ username }}</h3>
  </div>
  <hr>
  <div v-if="!showApp" class="container">
      <div class="left-panel">
        <div class="user-list-panel">
          <div class="list-header">
            <h3>사용자 목록</h3>
            <button @click="refreshUsers">새로고침</button>
          </div>
          <div v-if="userLoadErrorMessage" class="error">{{ userLoadErrorMessage }}</div>
          <div v-for="(username, idx) in userList" :key="idx" class="item">
            <h4>{{ username }}</h4>
          </div>
        </div>
        <div class="channel-name-panel">
          <div class="list-header">
            <h3>채팅방 채널</h3>
            <button @click="refreshChannels">새로고침</button>
          </div>
          <div v-if="channelLoadErrorMessage" class="error">{{ channelLoadErrorMessage }}</div>
          <div v-for="(channel, idx) in channelList" :key="idx" class="item">
            <h4>{{ channel.channelName }}</h4>
            <button v-if="currentChannelId !== channel.channelId" @click="subscribeToChannel(channel.channelId)">
              구독
            </button>
            <span v-else>입장 중</span>
          </div>
        </div>
      </div>
      <div class="middle-panel">
        <video id="localVideo" autoplay muted></video>
        <video id="remoteVideo" autoplay></video>
      </div>
      <div class="right-panel">
        <div class="message-input" v-if="currentChannelId">
            <input v-model="content" @keyup.enter="sendMessage" placeholder="메시지 입력" />
            <button @click="sendMessage">전송</button>
          </div>
        <div class="message-list">
          <div v-for="(message, idx) in messageList" :key="idx" class="message-item">
            <p><strong>{{ message.username }}:</strong> {{ message.content }}</p>
          </div>
      </div>
    </div>
    </div>
</template>

<script>
  import Peer from 'simple-peer';
  import Stomp from 'webstomp-client';
  import SockJS from 'sockjs-client';
  import axios from 'axios';

  export default {
    name: 'App',
    data() {
      return {
        username: '',
        currentChannelId: null,
        currentSubscriptionId: null,
        content:'',
        userList: [],
        channelList: [],
        messageList: [],
        stompClient: null,
        connected: false,
        showApp: true,
        connectionErrorMessage: '',
        userLoadErrorMessage: '',
        channelLoadErrorMessage: ''
      }; 
    },
    methods: {
      sendMessage() {
        if (this.content.trim() !== '') {
          this.send();
          this.content = "";
        }
      },
      send() {
        if (this.stompClient && this.connected) {
          const msg = {
            username: this.username,
            content: this.content
          };
          this.stompClient.send("/app/channels/" + this.currentChannelId + '/text/messages', JSON.stringify(msg), {});
        }
      },
      refreshUsers() {
        axios.get('http://localhost:8080/users')
        .then(response => {
          this.userLoadErrorMessage = '';
          const userListFromResponse = response.data
          .filter(user => user.username !== this.username)
          .map(user => user.username);
          this.userList = userListFromResponse;
        })
        .catch(() => {
          this.userLoadErrorMessage = 'Failed to load user list';
        });
      },
      refreshChannels() {
        axios.get('http://localhost:8080/channels')
          .then(response => {
            this.channelLoadErrorMessage = ''
            this.channelList = response.data;
          })
          .catch(() => {
            this.channelLoadErrorMessage = 'Failed to load channel list'
          });
      },
      subscribeToChannel(channelId) {
        if (this.currentChannelId) {
          this.stompClient.unsubscribe(this.currentSubscriptionId);
          this.messageList = [];
        }

        const subscription = this.stompClient.subscribe('/topic/channels/' + channelId + '/text', res => {
          const message = JSON.parse(res.body);
          this.messageList.push(message);
        });

        this.currentChannelId = channelId;
        this.currentSubscriptionId = subscription.id;
      },
      connect() {
        const serverURL = "http://localhost:8080/ws";
        let socket = new SockJS(serverURL);
        this.stompClient = Stomp.over(socket);

        const headers = {
          username: this.username,
        }

        // 서버에 웹소켓 연결 요청
        this.stompClient.connect(headers, 
          () => {
            this.connected = true;
            this.showApp = false;
            this.refreshUsers();
            this.refreshChannels();

            // 온라인 상태인 사용자 정보 구독
            this.stompClient.subscribe("/topic/users", res => {
              const message = JSON.parse(res.body);
              if (message.eventType === "CONNECT") {
                this.userList.push(message.username);
              }
              else {
                this.userList = this.userList.filter(username => username != message.username);
              }
            });
          },
          () => {
            this.connected = false;
            this.connectionErrorMessage = 'CONNECTION ERROR';
            this.showApp = true;
          }
        );
      }
    }
  }
</script>

<style>
  .container {
    display: flex;
    justify-content: space-between;
    gap: 20px;
  }

  .left-panel {
    flex: 1;
    display: flex;
    flex-direction: column;
    padding-right: 10px; /* 우측 패딩 추가 */
  }

  .middle-panel {
    flex: 1; /* 중간 패널을 동일한 너비로 설정 */
    display: flex;
    flex-direction: column;
    align-items: center; /* 내부 컨텐츠를 가운데 정렬 */
    padding: 0 10px; /* 좌우 패딩 추가 */
  }

  .right-panel {
    flex: 1;
    display: flex;
    flex-direction: column; /* 세로로 쌓이도록 설정 */
    padding-left: 10px; /* 좌측 패딩 추가 */
  }

  .list-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 10px; /* 아래쪽 마진 추가 */
  }

  .list-header h3 {
    margin: 0; /* 기본 마진 제거 */
  }

  .user-list-panel,
  .channel-name-panel,
  .message-list {
    flex: 1; /* 내용이 확장되도록 설정 */
    overflow-y: auto; /* 내용이 넘칠 때 스크롤 생성 */
  }

  .item {
    margin-bottom: 10px;
    padding: 5px 10px;
    border: 1px solid #ccc;
    border-radius: 5px;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  .error {
    color: red;
    margin-top: 10px;
  }

  .video-container {
    width: 100%; /* 비디오 컨테이너의 너비를 100%로 설정 */
    max-width: 400px; /* 최대 너비 지정 */
    margin-bottom: 20px; /* 아래쪽 마진 추가 */
  }

  .video-container video {
    width: 100%; /* 비디오 요소의 너비를 100%로 설정 */
    height: auto; /* 비디오 요소의 높이를 자동으로 조정 */
    display: block; /* 비디오 요소를 블록 요소로 표시하여 레이아웃에 맞춤 */
  }
</style>
