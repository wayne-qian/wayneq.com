<script setup lang="ts">
import { reactive, ref } from "vue";

const config = reactive({
    mapWidth: 0,
    mapHeight: 0,
    gridSize: 16,
    lives: new Map<string, void>(),
    gen: 0,
});

const stack = ref([] as Map<string, void>[]);

function mapStyle() {
    const ox = Math.floor(config.mapWidth / 2);
    const oy = Math.floor(config.mapHeight / 2);

    const dx = ox % config.gridSize
    const dy = oy % config.gridSize
    const lineColor = '#222'
    return {
        backgroundImage: `linear-gradient(90deg, transparent 0 ${dx}px,${lineColor} ${dx}px ${dx + 1}px, transparent ${dx + 1}px 100%),  linear-gradient( transparent 0 ${dy}px,${lineColor} ${dy}px ${dy + 1}px, transparent ${dy + 1}px 100%)`,
        backgroundSize: `${config.gridSize}px ${config.gridSize}px`,
    };
}
function oriStyle() {
    const ox = Math.floor(config.mapWidth / 2);
    const oy = Math.floor(config.mapHeight / 2);
    return {
        left: `${ox - 1}px`,
        top: `${oy - 1}px`
    }
}

function cellStyle(k: string) {
    const ox = Math.floor(config.mapWidth / 2);
    const oy = Math.floor(config.mapHeight / 2);

    const [sx, sy] = k.split(",");
    const x = parseInt(sx);
    const y = parseInt(sy);

    const left = ox + x * config.gridSize

    const top = oy + y * config.gridSize

    return {
        width: `${config.gridSize - 1}px`,
        height: `${config.gridSize - 1}px`,
        left: `${left + 1}px`,
        top: `${top + 1}px`,
    };
}

function arround(x: number, y: number, cb: (x: number, y: number) => void) {
    cb(x, y - 1);
    cb(x, y + 1);
    cb(x - 1, y - 1);
    cb(x - 1, y + 1);
    cb(x + 1, y - 1);
    cb(x + 1, y + 1);
    cb(x + 1, y);
    cb(x - 1, y);
}

function tap(ev: MouseEvent) {
    const ox = Math.floor(config.mapWidth / 2);
    const oy = Math.floor(config.mapHeight / 2);


    const el = document.getElementsByClassName("map")[0];
    const rc = el.getBoundingClientRect()

    const mx = ev.clientX - rc.left
    const my = ev.clientY - rc.top

    const x = Math.floor((mx - ox) / config.gridSize);

    const y = Math.floor((my - oy) / config.gridSize);

    const k = `${x},${y}`;
    const ll = new Map(config.lives);
    if (ll.has(k)) {
        ll.delete(k);
    } else {
        ll.set(k);
    }
    config.lives = ll;
}

function next() {
    config.gen++;
    const expand = new Map<string, void>();
    config.lives.forEach((_, i) => {
        expand.set(i);
        const [sx, sy] = i.split(",");
        const x = parseInt(sx);
        const y = parseInt(sy);
        arround(x, y, (x, y) => expand.set(`${x},${y}`));
    });

    const born = [] as Array<string>;
    const die = [] as Array<string>;

    expand.forEach((_, i) => {
        let n = 0;
        const [sx, sy] = i.split(",");
        const x = parseInt(sx);
        const y = parseInt(sy);
        arround(x, y, (x, y) => {
            if (config.lives.has(`${x},${y}`)) n++;
        });
        if (n < 2) die.push(i);
        if (n > 3) die.push(i);
        if (n == 3) born.push(i);
    });
    const newLives = new Map(config.lives);
    born.forEach((i) => newLives.set(i));
    die.forEach((i) => newLives.delete(i));
    stack.value.push(config.lives);
    if (stack.value.length > 16) {
        stack.value.shift();
    }
    config.lives = newLives;
}

function back() {
    config.gen--;
    if (stack.value.length > 0) {
        config.lives = stack.value[stack.value.length - 1];
        stack.value.pop();
    }
}

function zoomIn() {
    if (config.gridSize < 50) {
        config.gridSize += 4
    }
}

function zoomOut() {
    if (config.gridSize > 4) {
        config.gridSize -= 4
    }
}

function actualSize() {
    config.gridSize = 16
}

function clear() {
    config.gen = 0;
    stack.value = []
    config.lives = new Map()
}

function resize() {
    const el = document.getElementsByClassName("map")[0];
    config.mapWidth = el.clientWidth;
    config.mapHeight = el.clientHeight;
}

window.addEventListener("resize", resize);
window.addEventListener("load", resize);
</script>

<template >
    <div id="app" tabindex="0" @keydown.enter="next" style="background-color: black;">
        <div class="map" :style="mapStyle()" @click="tap">
            <div class="ori" :style="oriStyle()" />
            <div v-for="k in config.lives.keys()" :key="k" class="cell" :style="cellStyle(k)"></div>
        </div>
        <input class="btn" @click="zoomIn" value="Zoom in" type="button" />
        <input class="btn" @click="zoomOut" value="Zoom out" type="button" />
        <input class="btn" @click="actualSize" value="Actual size" type="button" /><span style="color: white"> gen: {{
            config.gen }}</span>
        <div style="position:absolute;bottom:0;width: 100%;">
            <input class="btn" @click="back" :disabled="stack.length == 0" value="Back" type="button" />
            <input class="btn" @click="next" value="Next" type="button" />
            <input class="btn" @click="clear" value="Clear" type="button" style="float: right" />
        </div>
    </div>
</template>

<style scoped>
.ori {
    width: 3px;
    height: 3px;
    background: #333;
    position: absolute;
}

.map {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}

.cell {
    position: absolute;
    background-color: white;
}

.btn {
    font-size: 1.5em;
    margin: 0.5em;
    touch-action: manipulation;
}
</style>
