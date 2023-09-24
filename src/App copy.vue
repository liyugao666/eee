<template>
  <div class="main">
    <div class="me">
      <video ref="me" id="me" autoplay playsinline></video>
      <button @click="createOffer">发起一个链接请求</button>
      <div> 
        左侧用户的sdp令牌
        <el-input v-model="offerSdp" />
      </div>
      <div>
        响应方的sdp
        <el-input v-model="answerSdp" />
        <el-button @click="addAnser">确认响应方的信息</el-button>
      </div>
     
    </div>


    
    <div class="you">
      <video ref="you" id="you" autoplay playsinline></video>
      <button @click="createAnwser"> 创建一个 回应的视频</button>
      <div>
          发起方的sdp <el-input v-model="comerSdp" />
      </div>
      <div> 
        回应方的 SDP
        <el-input v-model="answerSdp" />
      </div>
    </div>
    
  </div>
</template>
<script setup>
import { ref, onMounted } from 'vue';

// 自己的视频窗口
const me = ref(null);
// 对方的视频窗口
const you = ref(null)

// 自己的摄像头文件流
const meStream = ref(null);
// 打开自己的摄像头
const openCam = async ()=>{
    meStream.value = await navigator.mediaDevices.getUserMedia({
      audio: true,
      video: true
    });
    me.value.srcObject = meStream.value;
  

    // 添加本地流到 pc
    meStream.value.getTracks().forEach((track) => {
      pc.value.addTrack(track, meStream.value);
    });

    // ontrack 监听由新的视频内容的推送 onTrack是在对方给我视频信号的时候触发
    pc.value.ontrack = (event) => {
      console.log('视频在持续播放',event)
      you.value.srcObject = event.streams[0];
    };

}

const  offerSdp = ref('');
const answerSdp = ref('');

const createOffer = async () => {
  const offer = await pc.value.createOffer();
  await pc.value.setLocalDescription(offer); // 当前我自己的是一个消息的发起方 所以我的本地描述符 是offer
  // 监听 RTCPeerConnection 的 onicecandidate 事件，当 ICE 服务器返回一个新的候选地址时，就会触发该事件
  pc.value.onicecandidate = async (event) => {
    if (event.candidate) {
      offerSdp.value = JSON.stringify(pc.value.localDescription);
    }
  };
};

// 
const addAnser  = async ()=>{
  const answer = JSON.parse(answerSdp.value);
  pc.value.setRemoteDescription(answer)
}


// 作为回应方的摄像头文件流
const youSteam = ref(null)
const comerSdp = ref('');

const createAnwser = async ()=>{
  const offer = JSON.parse(comerSdp.value); 

  pc.value.onicecandidate = async (event) => {
    // Event that fires off when a new answer ICE candidate is created
    if (event.candidate) {
      answerSdp.value = JSON.stringify(pc.value.localDescription);
    }
  };
  await pc.value.setRemoteDescription(offer); // 当我是响应方法的是  我的Remote 就是刚才openCam的那个发起方的sdp信息
  const answer = await pc.value.createAnswer();
  await pc.value.setLocalDescription(answer); // 当我是响应方来点击createAnwser的时候我的 answer 就是本地

}



const pc = ref(null);// peerConnect的简写 这个变量是希望代指后面创建的RTC链接
const initConnect = ()=>{
    pc.value = new RTCPeerConnection({
      iceServers: [{ urls: "stun:stun.voipbuster.com " }],
    });

    openCam();
   
}


onMounted(initConnect);


</script>

<style>
.main{
  display: flex;
}
.me,.you {
  width: 400px;
  height: 500px;
  border: 1px solid #000;
}
#me,#you{
  width: 400px;
  height: 300px;
  border: 1px solid #000;
}
</style>