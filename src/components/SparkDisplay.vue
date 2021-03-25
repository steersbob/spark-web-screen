<script lang="ts">
/* eslint-disable no-console */
import {
  computed,
  defineComponent,
  nextTick,
  onBeforeUnmount,
  onMounted,
  Ref,
  ref,
} from 'vue';

export default defineComponent({
  name: 'SparkDisplay',
  setup: () => {
    const width = 320;
    const height = 240;

    const connected = ref(false);
    const hostname = ref('localhost');
    const port = ref(7376);
    const viewport: Ref<HTMLCanvasElement | null> = ref(null);

    let renderContext: CanvasRenderingContext2D | null = null;
    let ws: WebSocket | null = null;
    let buf: ArrayBuffer | null = null;
    let buf8: Uint8ClampedArray | null = null;
    let data: Uint32Array | null = null;
    let handle: number | undefined = undefined;
    let touchCount: number = 0;

    const url = computed(() => `ws://${hostname.value}:${port.value}`);

    const _autoconnect = ref(false);
    const autoconnect = computed({
      get: () => _autoconnect.value,
      set: v => {
        if (v !== _autoconnect.value) {
          v ? setupSocket() : closeSocket();
        }
        _autoconnect.value = v;
      },
    });

    function renderCanvas() {
      if (renderContext != null && buf8 != null) {
        const imagedata = renderContext.createImageData(width, height);
        imagedata.data.set(buf8);
        renderContext.putImageData(imagedata, 0, 0);
      }
    }

    function closeSocket() {
      if (ws) {
        ws.close();
        ws = null;
        connected.value = false;
      }
    }

    function setupSocket() {
      try {
        createSocket();
      }
      catch (e) {
        console.error(e);
        rescheduleSetup();
      }
    }

    function createSocket() {
      console.log('socket connecting...');
      ws = new WebSocket(url.value);
      ws.binaryType = 'arraybuffer';

      ws.onopen = () => {
        console.log('socket open');
        connected.value = true;
      };

      ws.onmessage = (msg: MessageEvent) => {
        handleScreenUpdate(new DataView(msg.data), msg.data.byteLength);
      };

      ws.onerror = () => {
        ws?.close();
      };

      ws.onclose = () => {
        console.log('socket closed');
        closeSocket();
        rescheduleSetup();
      };
    }

    function rescheduleSetup() {
      if (autoconnect.value) {
        setTimeout(() => {
          closeSocket();
          setupSocket();
        }, 1000);
      }
    }

    function handleScreenUpdate(buffer: DataView, length: number) {
      let index = 0;
      while (index < length && data != null) {
        const addr = buffer.getUint32(index, true);
        const color = buffer.getUint32(index + 4, true);

        const rr = ((color >>> 11) % 32);
        const gg = ((color >>> 5) % 64);
        const bb = ((color >>> 0) % 32);

        const r = (rr << 3) | ((rr >>> 2) & 7);
        const g = (gg << 2) | ((gg >>> 3) & 3);
        const b = (bb << 3) | ((bb >>> 2) & 7);

        data[addr] =
          (255 << 24) | // alpha
          (b << 16) | // blue
          (g << 8) | // green
          r; // red

        index += 8;
      }
    }

    function getMousePos(evt: MouseEvent): { x: number; y: number } {
      return viewport.value != null
        ? {
          x: evt.clientX - viewport.value.offsetLeft,
          y: evt.clientY - viewport.value.offsetTop,
        }
        : {
          x: 0,
          y: 0,
        };
    }

    function handleScreenTouch(evt: MouseEvent) {
      const { x, y } = getMousePos(evt);
      if (connected.value && ws != null) {
        console.log(`sending touch x=${x},y=${y},touchCount=${touchCount}`);
        const buf = new ArrayBuffer(5);
        const view = new DataView(buf);
        view.setInt8(0, touchCount ? 1 : 2); // command
        view.setUint16(1, x, true);
        view.setUint16(3, y, true);
        ws.send(buf);
      }
      else {
        console.log('no touch event sent - not connected');
      }
    }

    function onMouseDown(evt: MouseEvent) {
      console.log(`mousedown button=${evt.button}`);
      if (evt.button === 0) {
        touchCount += 1;
        handleScreenTouch(evt);
      }
    }

    function onMouseUp(evt: MouseEvent) {
      console.log(`mouseup button=${evt.button}`);
      if (evt.button === 0) {
        touchCount -= 1;
        handleScreenTouch(evt);
      }
    }

    function onMouseMove(evt: MouseEvent) {
      if (touchCount > 0) {
        handleScreenTouch(evt);
      }
    }

    onMounted(async () => {
      await nextTick();
      renderContext = viewport.value!.getContext('2d');

      if (renderContext) {
        buf = new ArrayBuffer(width * height * 4);
        buf8 = new Uint8ClampedArray(buf);
        data = new Uint32Array(buf);

        for (let i = 0; i < width * height; i += 1) {
          data[i] = 255 << 24; // initalize alpha to 255, leave black
        }
        renderCanvas();
      }

      handle = window.setInterval(renderCanvas, 100); // 10 frames per second max
    });

    onBeforeUnmount(() => {
      autoconnect.value = false;
      closeSocket();
      renderContext = null;
      window.clearInterval(handle);
      handle = undefined;
    });

    return {
      viewport,
      autoconnect,
      connected,
      hostname,
      port,
      width,
      height,
      onMouseDown,
      onMouseUp,
      onMouseMove,
    };
  },
});
</script>

<template>
  <div>
    <div class="input-container">
      <label>Hostname</label>
      <input v-model="hostname" :disabled="autoconnect">
      <label>Port</label>
      <input v-model="port" type="number" :disabled="autoconnect">
      <button @click="autoconnect = !autoconnect">
        {{ autoconnect ? 'Disconnect' : 'Connect' }}
      </button>
    </div>
    <div class="viewport-container">
      <canvas
        v-show="connected"
        ref="viewport"
        :width="width"
        :height="height"
        class="view"
        @mousedown="onMouseDown"
        @mouseup="onMouseUp"
        @mousemove="onMouseMove"
      />
      <div
        v-show="!connected"
        :width="width"
        :height="height"
        class="glass"
      />
    </div>
  </div>
</template>

<style scoped>
.input-container {
  display: flex;
  flex-flow: column;
  align-items: center;
  margin-bottom: 30px;
}
.input-container label {
  margin-bottom: 0.5em;
  margin-top: 1em;
}
.input-container button {
  margin-top: 1em;
}
.viewport-container {
  margin: 30px;
}
.display {
  margin: 0 auto;
}
.glass,
.view {
  display: block;
  margin: 0 auto;
  width: 320px;
  height: 240px;
}
.glass {
  background-color: rgba(255, 255, 255, 0.75);
  filter: blur(5px);
  display: none;
}
</style>
