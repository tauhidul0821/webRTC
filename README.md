# WebRTC Audio Call Conference with Angular

## 1. Project Setup
```sh
ng new angular-webrtc
cd angular-webrtc
npm install socket.io-client peerjs
```

## 2. Set Up Signaling Server (Node.js & Socket.io)
Create `server.js`:
```javascript
const io = require("socket.io")(3000, { cors: { origin: "*" } });
let users = {};

io.on("connection", (socket) => {
  socket.on("join", (roomId) => {
    socket.join(roomId);
    users[socket.id] = roomId;
    socket.to(roomId).emit("new-user", socket.id);
  });

  socket.on("signal", (data) => {
    io.to(data.to).emit("signal", { from: socket.id, signal: data.signal });
  });

  socket.on("disconnect", () => {
    const roomId = users[socket.id];
    socket.to(roomId).emit("user-disconnected", socket.id);
    delete users[socket.id];
  });
});
```
Run server:
```sh
node server.js
```

## 3. Implement WebRTC in Angular
### Create WebRTC Service (`webrtc.service.ts`)
```typescript
import { Injectable } from '@angular/core';
import { io } from 'socket.io-client';
import Peer from 'peerjs';

@Injectable({ providedIn: 'root' })
export class WebrtcService {
  private socket = io("http://localhost:3000");
  private peer: Peer;
  private peers: { [id: string]: any } = {};

  constructor() {
    this.peer = new Peer();
    this.peer.on("open", (id) => this.socket.emit("join", "conference-room"));
    this.socket.on("new-user", (userId) => this.callUser(userId));
    this.socket.on("signal", (data) => this.peer.signal(data.signal));
    this.socket.on("user-disconnected", (userId) => { if (this.peers[userId]) this.peers[userId].close(); });
  }

  startCall(stream: MediaStream) {
    this.peer.on("call", (call) => {
      call.answer(stream);
      call.on("stream", (remoteStream) => this.addAudioStream(remoteStream));
    });
  }

  callUser(userId: string) {
    navigator.mediaDevices.getUserMedia({ audio: true }).then((stream) => {
      const call = this.peer.call(userId, stream);
      call.on("stream", (remoteStream) => this.addAudioStream(remoteStream));
      this.peers[userId] = call;
    });
  }

  private addAudioStream(stream: MediaStream) {
    const audio = new Audio();
    audio.srcObject = stream;
    audio.autoplay = true;
    document.body.appendChild(audio);
  }
}
```

### Create Conference Component (`conference.component.ts`)
```typescript
import { Component, OnInit } from '@angular/core';
import { WebrtcService } from '../webrtc.service';

@Component({
  selector: 'app-conference',
  template: `<button (click)="startCall()">Join Audio Call</button>`
})
export class ConferenceComponent implements OnInit {
  constructor(private webrtcService: WebrtcService) {}
  ngOnInit(): void {}
  startCall() {
    navigator.mediaDevices.getUserMedia({ audio: true }).then((stream) => {
      this.webrtcService.startCall(stream);
    });
  }
}
```

## 4. Add to App Module (`app.module.ts`)
```typescript
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { AppComponent } from './app.component';
import { ConferenceComponent } from './conference/conference.component';

@NgModule({
  declarations: [AppComponent, ConferenceComponent],
  imports: [BrowserModule],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

## 5. Run Angular App
```sh
ng serve
```

## Next Steps
- Add UI for connected users.
- Implement mute/unmute functionality.
- Support video calling.
- Deploy the signaling server to a cloud service.
