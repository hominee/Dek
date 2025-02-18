<script setup lang="ts">
import { ref, reactive, onBeforeMount } from "vue";
import { invoke } from "@tauri-apps/api/tauri";
import { appWindow } from "@tauri-apps/api/window";

import {
  MouseStatus,
  WheelStatus,
  KeyboardStatus,
  MessageType,
  InputEventType,
} from "../common/Constans";
import { handleKeyboardEvent, handleMouseEvent } from "../common/InputEvent";

const PORT = 7856;
const data = reactive({
  account: {
    id: "",
    password: "",
  },
  receiverAccount: {
    id: "",
    password: "",
  },
  isShowRemoteDesktop: false,
});

const desktop = ref();

let ws: WebSocket;
let pc: RTCPeerConnection;
let dc: RTCDataChannel;
let webcamStream: any;

let remoteDesktopDpi: any;

onBeforeMount(async () => {
  data.account = await invoke("generate_account");
  initWebSocket();
});

/********************************* connect *************************************/

// we use websocket to heartbeat to the server 
// every 10 seconds
const initWebSocket = () => {
  ws = new WebSocket(`ws://127.0.0.1:${PORT}/conn/${data.account.id}`);

  ws.onopen = () => {
    setInterval(() => {
      sendToServer({
        msg_type: "heartbeat",
        receiver: "",
        sender: "",
        msg: "",
      });
    }, 1000 * 10);
  };

  ws.onmessage = async (e: any) => {
    let msg = JSON.parse(e.data);
    switch (msg.msg_type) {
      case MessageType.VIDEO_OFFER: // Invitation and offer to chat
        handleVideoOfferMsg(msg);
        break;
      case MessageType.VIDEO_ANSWER: // Callee has answered our offer
        handleVideoAnswerMsg(msg);
        break;
      case MessageType.NEW_ICE_CANDIDATE: // A new ICE candidate has been received
        handleNewICECandidateMsg(msg);
        break;
      case MessageType.REMOTE_DESKTOP:
        handleRemoteDesktopRequest(msg);
        break;
      case MessageType.CLOSE_REMOTE_DESKTOP:
        close();
        break;
    }
  };

  ws.onerror = () => {
    console.log("connection not established due to error");
  };
};

const handleVideoOfferMsg = async (msg: any) => {
  data.receiverAccount.id = msg.sender;

  await initRTCPeerConnection();

  let desc = new RTCSessionDescription(JSON.parse(msg.msg));
  await pc.setRemoteDescription(desc);

  await pc.setLocalDescription(await pc.createAnswer());
  sendToServer({
    msg_type: MessageType.VIDEO_ANSWER,
    receiver: data.receiverAccount.id,
    msg: JSON.stringify(pc.localDescription),
    sender: data.account.id,
  });
};

const handleVideoAnswerMsg = async (msg: any) => {
  let desc = new RTCSessionDescription(JSON.parse(msg.msg));
  await pc.setRemoteDescription(desc).catch(reportError);
};

const handleNewICECandidateMsg = async (msg: any) => {
  let candidate = new RTCIceCandidate(JSON.parse(msg.msg));
  try {
    await pc.addIceCandidate(candidate);
  } catch (err) {
    reportError(err);
  }
};

const handleRemoteDesktopRequest = async (msg: any) => {
  if (msg.msg != data.account.password) {
    console.log("password error!");
    return;
  }

  data.receiverAccount.id = msg.sender;

  await initRTCPeerConnection();

  initRTCDataChannel();

  // get local desktop
  webcamStream = await navigator.mediaDevices.getDisplayMedia({
    video: true,
    audio: false,
  });

  webcamStream.getTracks().forEach((track: any) =>
    // pc.addTransceiver(track, { streams: [webcamStream] })
    pc.addTrack(track, webcamStream)
  );

  sendOffer();
};

// webrtc
const initRTCPeerConnection = () => {
  // Create an RTCPeerConnection which knows to use our chosen
  // STUN server.
  const iceServer: object = {
    iceServers: [
      {
        url: "stun:stun.l.google.com:19302",
      },
      {
        url: "turn:numb.viagenie.ca",
        username: "webrtc@live.com",
        credential: "muazkh",
      },
    ],
  };

  pc = new RTCPeerConnection(iceServer);

  // Set up event handlers for the ICE negotiation process.
  pc.onicecandidate = handleICECandidateEvent;
  pc.oniceconnectionstatechange = handleICEConnectionStateChangeEvent;
  pc.onicegatheringstatechange = handleICEGatheringStateChangeEvent;
  pc.onsignalingstatechange = handleSignalingStateChangeEvent;
  // pc.onnegotiationneeded = handleNegotiationNeededEvent;
  pc.ontrack = handleTrackEvent;
  pc.ondatachannel = handleDataChannel;
};

const handleICECandidateEvent = (event: any) => {
  if (event.candidate) {
    sendToServer({
      msg_type: MessageType.NEW_ICE_CANDIDATE,
      receiver: data.receiverAccount.id,
      msg: JSON.stringify(event.candidate),
      sender: data.account.id,
    });
  }
};

const handleICEConnectionStateChangeEvent = (event: any) => {
  console.log("*** ICE连接状态变为" + pc.iceConnectionState);
};

const handleICEGatheringStateChangeEvent = (event: any) => {
  console.log("*** ICE聚集状态变为" + pc.iceGatheringState);
};

const handleSignalingStateChangeEvent = (event: any) => {
  console.log("*** WebRTC信令状态变为: " + pc.signalingState);
};

// get data stream
const handleTrackEvent = (event: any) => {
  desktop.value.srcObject = event.streams[0];

  document.onkeydown = (e) => {
    sendToClient({
      type: InputEventType.KEY_EVENT,
      data: {
        eventType: KeyboardStatus.MOUSE_DOWN,
        key: e.key,
      },
    });
  };

  document.onkeyup = (e) => {
    sendToClient({
      type: InputEventType.KEY_EVENT,
      data: {
        eventType: KeyboardStatus.MOUSE_UP,
        key: e.key,
      },
    });
  };
};

// p2p data channel
const handleDataChannel = (e: any) => {
  dc = e.channel;
  dc.onopen = () => {
    console.log("datachannel open");
  };

  dc.onmessage = (event: any) => {
    remoteDesktopDpi = JSON.parse(event.data);
  };

  dc.onclose = () => {
    console.log("datachannel close");
  };
};

// webrtc data channel
const initRTCDataChannel = () => {
  dc = pc.createDataChannel("my channel", {
    ordered: true,
  });

  dc.onopen = () => {
    console.log("datachannel open");
    dc.send(
      JSON.stringify({
        width: window.screen.width * window.devicePixelRatio,
        height: window.screen.height * window.devicePixelRatio,
      })
    );
  };

  dc.onmessage = (event: any) => {
    let msg = JSON.parse(event.data);
    switch (msg.type) {
      case InputEventType.MOUSE_EVENT:
        handleMouseEvent(msg.data);
        break;
      case InputEventType.KEY_EVENT:
        handleKeyboardEvent(msg.data);
        break;
    }
  };

  dc.onclose = () => {
    console.log("datachannel close");
  };
};

const sendOffer = async () => {
  // send offer
  const offer = await pc.createOffer();

  // Establish the offer as the local peer's current
  // description.
  await pc.setLocalDescription(offer);

  // Send the offer to the remote peer.
  sendToServer({
    msg_type: MessageType.VIDEO_OFFER,
    receiver: data.receiverAccount.id,
    msg: JSON.stringify(pc.localDescription),
    sender: data.account.id,
  });
};

/********************************* user event *************************************/

// request
const remoteDesktop = async () => {
  appWindow.setFullscreen(true);

  // show panel
  data.isShowRemoteDesktop = true;

  sendToServer({
    msg_type: MessageType.REMOTE_DESKTOP,
    receiver: data.receiverAccount.id,
    msg: data.receiverAccount.password,
    sender: data.account.id,
  });
};

const closeRemoteDesktop = async () => {
  appWindow.setFullscreen(false);
  data.isShowRemoteDesktop = false;

  close();

  sendToServer({
    msg_type: MessageType.CLOSE_REMOTE_DESKTOP,
    receiver: data.receiverAccount.id,
    msg: data.receiverAccount.password,
    sender: data.account.id,
  });
};

const mouseDown = (e: any) => {
  sendMouseEvent(e.x, e.y, mouseType(MouseStatus.MOUSE_DOWN, e.button));
};

const mouseUp = (e: any) => {
  sendMouseEvent(e.x, e.y, mouseType(MouseStatus.MOUSE_UP, e.button));
};

const wheel = (e: any) => {
  let type = e.wheelDelta > 0 ? WheelStatus.WHEEL_UP : WheelStatus.WHEEL_DOWN;
  sendMouseEvent(e.x, e.y, type);
};

const mouseMove = (e: any) => {
  sendMouseEvent(e.x, e.y, MouseStatus.MOUSE_MOVE);
};

const rightClick = (e: any) => {
  sendMouseEvent(e.x, e.y, MouseStatus.RIGHT_CLICK);
};

/********************************* common *************************************/

// send event
const sendMouseEvent = (x: number, y: number, eventType: string) => {
  if (remoteDesktopDpi) {
    let widthRatio = remoteDesktopDpi.width / desktop.value.clientWidth;
    let heightRatio = remoteDesktopDpi.height / desktop.value.clientHeight;

    let data = {
      x: parseInt((x * widthRatio).toFixed(0)),
      y: parseInt((y * heightRatio).toFixed(0)),
      eventType: eventType,
    };
    sendToClient({
      type: InputEventType.MOUSE_EVENT,
      data: data,
    });
  }
};

// get type
const mouseType = (mouseStatus: MouseStatus, button: number) => {
  let type = "";
  switch (button) {
    case 0:
      type = "left-" + mouseStatus;
      break;
    case 2:
      type = "right-" + mouseStatus;
      break;
    // TODO more button
  }

  return type;
};

const close = () => {
  // TODO authentication

  if (desktop.value.srcObject) {
    let tracks = desktop.value.srcObject.getTracks();
    tracks.forEach((track: any) => track.stop());
    desktop.value.srcObject = null;
  } else {
    webcamStream.getTracks().forEach((track: any) => track.stop());
  }

  // Close the peer connection
  pc.close();
};

const sendToServer = (msg: any) => {
  let msgJSON = JSON.stringify(msg);
  ws.send(msgJSON);
};

const sendToClient = (msg: any) => {
  let msgJSON = JSON.stringify(msg);
  dc.readyState == "open" && dc.send(msgJSON);
};
</script>

<template>
  <div class="sidebar">
    <div >
      <p>
        id: <span class="content-span">{{ data.account.id }}</span>
      </p>
      <p>
        password: <span class="content-span" >{{ data.account.password }}</span>
      </p>
    </div>
  </div>
  <div class="form">
    <input
      v-model="data.receiverAccount.id"
      type="text"
      placeholder="please input the id of responder"
    />
    <input
      v-model="data.receiverAccount.password"
      type="text"
      placeholder="please input the password of responder"
    />
    <button style="width: 100px; height: 50px; font-size: 20px" @click="remoteDesktop()">Start</button>
		<div style="height: 60px;">
		</div>
  </div>
  <video
    v-show="data.isShowRemoteDesktop"
    @mousedown="mouseDown($event)"
    @mouseup="mouseUp($event)"
    @mousemove="mouseMove($event)"
    @wheel="wheel($event)"
    @contextmenu.prevent="rightClick($event)"
    class="desktop"
    ref="desktop"
    autoplay
  ></video>
  <button
    v-if="data.isShowRemoteDesktop"
    class="close-btn"
		style="font-size: 16px"
    @click="closeRemoteDesktop()"
  >
    Close
  </button>
</template>

<style lang="less" scoped>
.sidebar {
  width: 100%;
  height: 160px;
  background: #121212;
  color: #fafafa;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  border-bottom: 1px solid #252525;
  box-sizing: border-box;
  > div {
    background: #121212;
    padding: 10px 20px;
    border-radius: 10px;
    p {
      line-height: 28px;
      font-size: 16px;
      span {
        font-size: 18px;
        font-weight: 600;
      }
    }
  }
}

.content-span {
	/*background-color : #B6B7C6;*/

}

.form {
  height: calc(100% - 160px);
  display: flex;
  flex-direction: column;
	justify-content: center;
  align-items: center;
  background: #121212;
  button {
    width: 280px;
    height: 34px;
    background: #11c111;
  }
	input[type="text"], textarea {
		background-color : #B6B7C6;
	}
}

input {
  width: 280px;
  outline: none;
  border: 1px solid #252525;
  padding: 0 10px;
  height: 34px;
  box-sizing: border-box;
  border-radius: 5px;
  margin-bottom: 30px;
}

button {
  outline: none;
  border: none;
  color: #fff;
  border-radius: 5px;
}

.desktop {
  width: 100%;
  height: 100%;
  position: fixed;
  top: 0;
  left: 0;
  background: #121212;
  cursor: none;
}

.close-btn {
  width: 70px;
  height: 40px;
  position: fixed;
  right: calc((100% - 40px) * 0.5 );
  bottom: 30px;
  z-index: 1;
  background: #d71526;
  font-size: 12px;
}
</style>
