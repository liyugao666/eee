<template>
    <div class="main">
    <div class="me">
      <video ref="me" id="me" autoplay playsinline></video>
      <button @click="takeAPhone">发起一个链接请求</button>

    </div>
    
    <div class="you">
      <video ref="you" id="you" autoplay playsinline></video>
      <button @click="reciPhone" v-show="showReci"> 接视频</button>
    </div>
    
  </div>
</template>
<script setup>
import { io } from "socket.io-client";
import { onMounted,ref } from "vue";
const pc = ref(null);
const meStream = ref(null);

const sc = io('http://127.0.0.1:3000')
sc.on('connect', ()=>{
  console.log('与socket服务器通讯成功')
})

const offerSDP = ref(null)
sc.on('tob',(res)=>{
  console.log('有人想我发起请求了',res)
  offerSDP.value = JSON.parse(res.offerSdp);
  showReci.value = true
})

sc.on('toa',res=>{
  if(!isOffer.value) return false ; // 只有发起发才需要处理对方的响应
  console.log('对方已经同意',res)
  pc.value.setRemoteDescription(JSON.parse(res.ansSdp))
})

const init = ()=>{
   pc.value = new RTCPeerConnection();
  openCam();
}


const isOffer = ref(false)


const takeAPhone = ()=>{
  isOffer.value = true;
  const offer = pc.value.createOffer();// 创建一个offer的 对象
  // 监听onicecandidate 事件 在链接成功之后会返回一个sdp信息
  // sdp对象就是一个描述当前的通讯设备的地址和内容的一个对象
  pc.value.onicecandidate = async (event) => {
    if (event.candidate) {
      const offerSdp = JSON.stringify(pc.value.localDescription);
      sc.emit('tob', { offerSdp })
    }
  };
  pc.value.setLocalDescription(offer); // 确认播放电话的就是 发起方

}

const me = ref(null);
const you = ref(null)
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

onMounted(init);

const showReci = ref(false)
const reciPhone = async ()=>{


  pc.value.onicecandidate = async (event) => {
    if (event.candidate) {
      const ansSdp = JSON.stringify(pc.value.localDescription);
      sc.emit('toa', { ansSdp })
    }
  };

  await pc.value.setRemoteDescription(offerSDP.value); // 当我是响应方法的是  我的Remote 就是刚才openCam的那个发起方的sdp信息
  const ans  = await pc.value.createAnswer();
  await pc.value.setLocalDescription(ans)

}


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