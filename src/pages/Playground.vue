<template>
    <div class="playground-root p-4 md:p-6 space-y-4 container mx-auto">
        <div class="playground-panel">
            <div class="flex flex-wrap items-center justify-between gap-3">
                <div>
                    <h2 class="text-xl font-semibold text-slate-100">RPC Playground</h2>
                    <p class="text-sm text-slate-400 mt-1">Test Discord activity events and inspect runtime logs.</p>
                </div>
                <button class="playground-primary-button" @click="discordTest">
                    Discord Test (connected: {{ isConnected }})
                </button>
            </div>
        </div>

        <!-- Logs Section -->
        <div class="playground-panel text-slate-300">
            <div class="flex items-center justify-between mb-2 gap-2">
                <h2 class="text-lg font-semibold mb-2 text-slate-100">Logs</h2>
                <button class="playground-secondary-button text-sm" @click="clearLogs">Clear Logs</button>
            </div>

            <div class="max-h-64 overflow-y-auto p-2 rounded-xl bg-slate-950/40 border border-white/10">
                <!-- <div v-if="logs.length === 0" class="text-gray-400">No logs available.</div>
                <ul v-else class="list-none">
                    <li v-for="(log, index) in logs" :key="index" class="text-sm text-gray-400">{{ log }}</li>
                </ul> -->
                <div v-if="logs.length === 0" class="text-slate-400">No logs available.</div>
                <ul v-else class="list-none">
                    <li v-for="(log, index) in logs" :key="index" class="text-sm py-0.5">
                        <span class="text-slate-500">[{{ new Date(log.timestamp).toLocaleString() }}]</span>
                        <span :class="{
                            'text-blue-400': log.type === 'info',
                            'text-red-400': log.type === 'error',
                            'text-yellow-400': log.type === 'warning',
                            'text-green-400': log.type === 'debug'
                        }">
                            [{{ log.type.toUpperCase() }}]
                        </span>
                        <span class="ml-1">{{ log.message }}</span>
                    </li>
                </ul>
            </div>


        </div>
    </div>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue';
import { invoke } from '@tauri-apps/api/core';
import { emit } from '@tauri-apps/api/event';
import { useGlobalState } from '@/composables/app-state';

const ActivityKind = {
    Playing: 0,
    Listening: 2,
    Watching: 3,
    Competing: 5
} as const;

const isConnected = ref(false);

const { logs, addLog, clearLogs } = useGlobalState();

function discordTest() {

    const appIdCode = '1361728268088381706';

    if (isConnected.value) {
        console.log('Disconnecting from Discord');
        emit('event_disconnect');
        isConnected.value = false;
        return;
    }

    invoke('connect_to_discord_rpc_3', {
        activity_json: JSON.stringify({
            app_id: appIdCode,
            details: 'Jhabol',
            // details: 'xmonad -> dwm -> spectrwm -> i3 -> bspwm -> qtile -> hyrpland -> xfce -> gnome -> sway',
            state: "/jhabol",
            activity_kind: ActivityKind.Watching,
            timestamp: createAgoTimestamp('1h 30m')
        }),
        action: 'connect',
    });
    isConnected.value = true;
}

// function to create timestamp behind current time.
// example: input is `4h 30m` means timestamp should start from 4 hours and 30 minutes behind current time.
function createAgoTimestamp(input: string) {
    const time = input.split(' ');
    let hours = 0;
    let minutes = 0;

    for (let i = 0; i < time.length; i++) {
        if (time[i].includes('h')) {
            hours = parseInt(time[i]);
        } else if (time[i].includes('m')) {
            minutes = parseInt(time[i]);
        }
    }

    const date = new Date();
    date.setHours(date.getHours() - hours);
    date.setMinutes(date.getMinutes() - minutes);

    return Math.floor(date.getTime() / 1000);
}


onMounted(() => {

})

</script>

<style scoped>
@reference "../theme/style.css";

.playground-root {
    position: relative;
}

.playground-panel {
    border: 1px solid rgba(148, 163, 184, 0.22);
    border-radius: var(--radius-lg);
    background:
        linear-gradient(150deg, rgba(124, 58, 237, 0.08), transparent 42%),
        rgba(13, 21, 38, 0.86);
    box-shadow: var(--shadow-soft);
    padding: 1rem;
}

.playground-primary-button {
    border-radius: 12px;
    border: 1px solid rgba(124, 58, 237, 0.45);
    color: #f8fafc;
    font-weight: 600;
    font-size: 0.88rem;
    padding: 0.65rem 1rem;
    background: linear-gradient(110deg, #7c3aed 0%, #22d3ee 100%);
    transition: transform var(--trans-fast), box-shadow var(--trans-fast), filter var(--trans-fast);
}

.playground-primary-button:hover {
    transform: translateY(-1px) scale(1.01);
    box-shadow: 0 0 24px rgba(124, 58, 237, 0.34);
    filter: brightness(1.03);
}

.playground-secondary-button {
    border-radius: 10px;
    border: 1px solid rgba(148, 163, 184, 0.35);
    color: #cbd5e1;
    background: rgba(30, 41, 59, 0.8);
    font-weight: 600;
    padding: 0.4rem 0.75rem;
    transition: all var(--trans-fast);
}

.playground-secondary-button:hover {
    color: #f8fafc;
    border-color: rgba(34, 211, 238, 0.5);
}
</style>
