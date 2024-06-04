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
          <div v-for="(username, idx) in userList" :key="idx" class="item">
            <h4>{{ username }}</h4>
          </div>
        </div>
        <div class="channel-name-panel">
          <div class="list-header">
            <h3>채팅방 채널</h3>
            <button @click="refreshChannels">새로고침</button>
          </div>
          <div v-for="(channel, idx) in channelList" :key="idx" class="item">
            <h4>{{ channel.channelName }}</h4>
            <button v-if="currentChannelId !== channel.channelId" @click="subscribeChannel(channel.channelId)">
              구독
            </button>
            <span v-else>입장 중</span>
          </div>
        </div>
      </div>
      <div class="middle-panel" v-if="currentChannelId">
        <div v-if="!videoOn">
          <button @click="joinVideoChannel">화상 채팅 참여</button>
        </div>
        <div v-else>
          <button @click="leaveVideoChannel">화상 채팅 나가기</button>
          <button @click="handleCamera">카메라 ON/OFF</button>
          <video ref="localVideo" autoplay></video>
          <video ref="remoteVideo" autoplay></video>
        </div>
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
  import Stomp from 'webstomp-client';
  import SockJS from 'sockjs-client';
  import axios from 'axios';

  export default {
    name: 'App',
    data() {
      return {
        username: '',
        currentChannelId: null,
        textSubscriptionId: null,
        content:'',
        userList: [],
        channelList: [],
        messageList: [],
        stompClient: null,
        showApp: true,
        connectionErrorMessage: '',
        localStream: null,
        remoteStream:  null,
        peerConnection: null,
        videoSubscriptionId: null,
        videoOn: false,
        peername: null,
        videoChannelSubscribeId: null,
        offerSubscribeId : null,
        answerSubscribeId: null,
        candidateSubscribeId: null
      }; 
    },
    methods: {
      async joinVideoChannel() {
        this.videoOn = true;
        try {
          this.localStream = await navigator.mediaDevices.getUserMedia({
            video: true,
            audio: true
          });

          this.localStream.getVideoTracks().forEach((track) => {
            track.enabled = false;
          });

          this.$refs.localVideo.srcObject = this.localStream;

          const videoChannelSubscribe = this.stompClient.subscribe('/topic/channels/' + this.currentChannelId +'/video', this.handleRequest);
          const offerSubscribe = this.stompClient.subscribe('/topic/offer/' + this.username, this.handleOffer);
          const answerSubscribe = this.stompClient.subscribe('/topic/answer/' + this.username, this.handleAnswer);
          const candidateSubscribe = this.stompClient.subscribe('/topic/candidate/' + this.username, this.handleCandidate);

          this.videoChannelSubscribeId = videoChannelSubscribe.id;
          this.offerSubscribeId = offerSubscribe.id;
          this.answerSubscribeId = answerSubscribe.id;
          this.candidateSubscribeId = candidateSubscribe.id;

          this.peerConnection = new RTCPeerConnection();
          this.localStream.getTracks().forEach(track => this.peerConnection.addTrack(track, this.localStream));

          this.peerConnection.onicecandidate = event => {
            if (event.candidate) {
              this.stompClient.send('/app/candidate', JSON.stringify({
                sender: this.username,
                receiver: this.peername,
                candidate: event.candidate
              }), {});
            }
          };

          this.peerConnection.ontrack = event => {
            if (this.$refs.remoteVideo.srcObject !== event.streams[0]) {
              this.$refs.remoteVideo.srcObject = event.streams[0];
            }
          };

          this.stompClient.send('/app/channels/' + this.currentChannelId + '/video/conn', JSON.stringify({
            username: this.username,
            eventType: 'CONNECT'
          }), {});
        } catch (error) {
          console.error('Error starting call', error);
        }
      },
      async handleRequest(message) {
        const peername = JSON.parse(message.body).username;
        this.peername = peername;
        if (peername !== this.username) {
          if (JSON.parse(message.body).eventType === "CONNECT") {
            const offer = await this.peerConnection.createOffer();
            this.peerConnection.setLocalDescription(offer);
            this.stompClient.send('/app/offer', JSON.stringify({
              sender: this.username,
              receiver: peername,
              signal: offer
            }), {});
          }
          else {
            this.peername = null;

          }
        }
      },
      async handleOffer(message) {
        const offer = JSON.parse(message.body).signal;
        this.peername = JSON.parse(message.body).sender;
        this.peerConnection.setRemoteDescription(new RTCSessionDescription(offer));
        const answer = await this.peerConnection.createAnswer();
        this.peerConnection.setLocalDescription(answer);
        this.stompClient.send('/app/answer', JSON.stringify({
          sender: this.username,
          receiver: JSON.parse(message.body).sender,
          signal: answer
        }), {});
      },
      async handleAnswer(message) {
        const answer = JSON.parse(message.body).signal;
        this.peerConnection.setRemoteDescription(new RTCSessionDescription(answer));
      },
      async handleCandidate(message) {
        const candidate = JSON.parse(message.body).candidate;
        await this.peerConnection.addIceCandidate(new RTCIceCandidate(candidate));
      },
      handleDisconnectedUser() {
        this.peername = null;

        if (this.peerConnection) {
          this.peerConnection.close();
          this.peerConnection = null;
        }
        if (this.remoteStream) {
          this.remoteStream.getTracks().forEach(track => track.stop());
          this.remoteStream = null;
          this.$refs.remoteVideo.srcObject = null;
        }
      },
      leaveVideoChannel() {
        this.peername = null;

        if (this.peerConnection) {
          this.peerConnection.close();
          this.peerConnection = null;
        }
        if (this.localStream) {
          this.localStream.getTracks().forEach(track => track.stop());
          this.localStream = null;
          this.$refs.localVideo.srcObject = null;
        }
        if (this.remoteStream) {
          this.remoteStream.getTracks().forEach(track => track.stop());
          this.remoteStream = null;
          this.$refs.remoteVideo.srcObject = null;
        }

        this.stompClient.unsubscribe(this.videoChannelSubscribeId);
        this.stompClient.unsubscribe(this.offerSubscribeId);
        this.stompClient.unsubscribe(this.answerSubscribeId);
        this.stompClient.unsubscribe(this.candidateSubscribeId);

        this.stompClient.send('/app/channels/' + this.currentChannelId + '/video/conn', JSON.stringify({
            username: this.username,
            eventType: 'DISCONNECT'
          }), {});

        this.videoOn = false;
      },
      handleCamera(){
        this.localStream.getVideoTracks().forEach((track) => {
          track.enabled = !track.enabled;
        });
      },
      connect() {
        const serverURL = "http://localhost:8080/ws";
        let socket = new SockJS(serverURL);
        this.stompClient = Stomp.over(socket);

        this.stompClient.connect({ username: this.username }, 
          () => {
            this.showApp = false;
            this.refreshUsers();
            this.refreshChannels();
            this.onlineUserSubscribe();
          },
          () => {
            this.connectionErrorMessage = 'CONNECTION ERROR';
            this.showApp = true;
          }
        );
      },
      refreshUsers() {
        axios.get('http://localhost:8080/users')
        .then(response => {
          const userListFromResponse = response.data
          .filter(user => user.username !== this.username)
          .map(user => user.username);
          this.userList = userListFromResponse;
        });
      },
      refreshChannels() {
        axios.get('http://localhost:8080/channels')
          .then(response => {
            this.channelList = response.data;
        });
      },
      onlineUserSubscribe() {
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
      subscribeChannel(channelId) {
        if (this.currentChannelId) {
          this.stompClient.unsubscribe(this.textSubscriptionId);
          // this.stompClient.unsubscribe(this.videoSubscriptionId);
          this.messageList = [];
        }

        const subscription = this.stompClient.subscribe('/topic/channels/' + channelId + '/text', res => {
          const message = JSON.parse(res.body);
          this.messageList.push(message);
        });

        this.currentChannelId = channelId;
        this.textSubscriptionId = subscription.id;
      },
      sendMessage() {
        if (this.content.trim() !== '') {
          this.stompClient.send("/app/channels/" + this.currentChannelId + '/text/messages', JSON.stringify({
            username: this.username,
            content: this.content
          }), {});
        }
          this.content = "";
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
