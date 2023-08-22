<template>
    <div id="terminal" ref="terminalRef"></div>
</template>
<script setup>
import { Terminal } from 'xterm'
import { FitAddon } from 'xterm-addon-fit'
import { AttachAddon } from 'xterm-addon-attach'
import { CanvasAddon } from 'xterm-addon-canvas'
import 'xterm/css/xterm.css'
import { onMounted, ref } from 'vue'
const term = ref()
let socket
const ROWS = 40
const COLS = 10
const terminalRef = ref(null)
const addons = {
    attach: { name: 'attach', ctor: AttachAddon, canChange: false },
    fit: { name: 'fit', ctor: FitAddon, canChange: false },
    canvas: { name: 'canvas', ctor: CanvasAddon, canChange: true },
}
const createXterm = () => {
    term.value = new Terminal({
        scrollback: 99999,
        fontSize: 14,
        lineHeight: 1.2,
        allowProposedApi: true,
        rows: ROWS, //行数
        cols: COLS, // 不指定行数，自动回车后光标从下一行开始
        convertEol: false, //启用时，光标将设置为下一行的开头
        disableStdin: true, //是否应禁用输入
        tabStopWidth: 4,
        // cursorStyle: "underline", //光标样式
        cursorBlink: true, //光标闪烁
        screenReaderMode: true,
        fontFamily: "Monaco, Menlo, Consolas, 'Courier New', monospace",
        theme: {
            foreground: '#ECECEC', //字体
            background: '#000000', //背景色
            cursor: 'help', //设置光标
            lineHeight: 10,
        },
    })
    term.value.open(terminalRef.value)
    addons.fit.instance = new FitAddon()
    addons.canvas.instance = new CanvasAddon()
    term.value.loadAddon(addons.canvas.instance)
    term.value.loadAddon(addons.fit.instance)
    addons.fit.instance.fit()
    const resizeScreen = () => {
        try {
            addons.fit.instance.fit()
        } catch (e) {
            console.log('e', e.message)
        }
    }
    window.addEventListener('resize', resizeScreen)
    initWebsocket()
}
const initWebsocket = () => {
    setTimeout(async() => {
      const res = await fetch('api/terminals?cols=' + ROWS + '&rows=' + COLS, { method: 'POST' });
      const processId = await res.text();
      let socketURL = 'ws://127.0.0.1:3000/terminals/';
      socketURL += processId;
      socket = new WebSocket(socketURL);
        socket.onopen = runRealTerminal
        socket.onclose = runFakeTerminal
        socket.onerror = runFakeTerminal
    }, 0)
}
const runRealTerminal = () => {
    addons.attach.instance = new AttachAddon(socket)
    term.value.loadAddon(addons.attach.instance)
    term.value.options['disableStdin'] = false
    // term.value.focus()
}
// const initialize = () => {
//     createXterm()
// }
// const getNewSession = () => {
//     initialize()
//     initWebsocket()
// }
const runFakeTerminal = () => {
    term.value.writeln('end')
    term.value.options['disableStdin'] = true
    term.value.dispose()
    addons.attach.instance = undefined
}
onMounted(() => {
    if (terminalRef.value) {
        console.log(terminalRef.value)
        createXterm()
    }
})
</script>
<style scoped>
#terminal {
    width: 100%;
    height: 100%;
}
</style>
