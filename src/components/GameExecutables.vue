<template>
    <div class="text-slate-300">
        <h3 class="text-sm text-slate-400">
            The game has multiple platform executables. Please select one to launch:
        </h3>

        <div class="text-xs mt-3 space-y-2.5">
            <div v-for="(executable) in filteredExecutables" :key="executable.name"
                class="exe-row">
                <div class="w-14 max-w-[80px]">
                    <div class="exe-os-pill">
                        {{ executable.os }}
                    </div>
                </div>

                <!-- Sections / Breadcrumbs must fade when too long -->
                <div class="relative overflow-hidden ">
                    <div class="flex flex-nowrap overflow-x-auto scrollbar-none max-w-full pr-4 fade-right">
                        <div v-for="(section, i) in splitExecutableName(executable)" :key="i"
                            class="exe-path-chip">
                            <span>{{ section }}</span>
                        </div>
                    </div>
                </div>

                <div class="justify-self-end">
                    <button class="exe-action-button"
                    :class="[
                        {
                            'exe-action-button-play': !gameActions?.isExecutableRunning(executable),
                            'exe-action-button-stop': gameActions?.isExecutableRunning(executable),
                        },
                    ]"
                        @click="handleLaunch(executable)"
                    >
                        {{ gameActions?.isExecutableRunning(executable) ? 'Stop' : 'Play' }}
                    </button>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">
import { EXECUTABLE_OS, GameActionsKey } from '@/constants/constants';
import { GameActionsProvider, type Game, type GameExecutable } from '@/types/types';
import { path, app } from '@tauri-apps/api';
import { computed, inject } from 'vue';

const props = defineProps<{
    game: Game
}>();

const emit = defineEmits<{
    play: [{game: Game, executable: GameExecutable}]
    stop: [{game: Game, executable: GameExecutable}]
    install_and_play: [{game: Game, executable: GameExecutable}]
}>();

const gameActions = inject<GameActionsProvider>(GameActionsKey);

const filteredExecutables = computed(() => {
    return props.game.executables.filter(executable => {
        // currently no support for linux and darwin
        return executable.os !== EXECUTABLE_OS.LINUX && executable.os !== EXECUTABLE_OS.DARWIN
            && !isValidPath(executable.name);
    });
});

function splitExecutableName(executable: GameExecutable) {
    const allSections = executable.name.split(/\\|\//);
    
    const last = executable.name.split(/\\|\//).pop();
    // remove file extension if there was none, just return the last section
    const name = last?.split('.').slice(0, -1).join('.') || last;
    return [
        ...allSections.slice(0, -1),
        name,
    ];
}

function getExecutablePath(executable: GameExecutable) {
    const allSections = executable.name.split(/\\|\//);
    const last = executable.name.split(/\\|\//).pop();
    // remove file extension if there was none, just return the last section
    const name = last?.split('.').slice(0, -1).join('.') || last;
    return [
        ...allSections.slice(0, -1)
    ].join(path.sep())
}

function getFilename(executable: GameExecutable) {
    const last = executable.name.split(/\\|\//).pop();
    // remove file extension if there was none, just return the last section
    return last;
}

function isValidPath(path: string) {
    const illegalChars = ['>', '<', ':', '"', '|', '?', '*'];
    return illegalChars.some(char => path.includes(char));
}

function handleLaunch(executable: GameExecutable) {
    // Handle the launch logic here
    console.log('Launching game:', props.game);
    if(executable.is_running) {
        emit('stop', {
            game: props.game,
            executable: {
                path: getExecutablePath(executable),
                segments: splitExecutableName(executable).length,
                filename: getFilename(executable),
                ...executable
            },
        });
    } else {
        if (!gameActions?.isGameExecutableInstalled(executable)) {
            emit('install_and_play', {
                game: props.game,
                executable: {
                    path: getExecutablePath(executable),
                    segments: splitExecutableName(executable).length,
                    filename: getFilename(executable),
                    ...executable
                },
            });
        } else {
            emit('play', {
                game: props.game,
                executable: {
                    path: getExecutablePath(executable),
                    segments: splitExecutableName(executable).length,
                    filename: getFilename(executable),
                    ...executable
                },
            });
        }
     
    }
    
}

</script>

<style scoped>
@reference "../theme/style.css";

.exe-row {
    display: grid;
    grid-template-columns: auto 1fr auto;
    gap: 0.6rem;
    align-items: center;
    width: 100%;
    border: 1px solid rgba(148, 163, 184, 0.2);
    border-radius: 12px;
    background: rgba(15, 23, 42, 0.56);
    padding: 0.55rem 0.6rem;
    transition: border-color var(--trans-fast), transform var(--trans-fast), background var(--trans-fast);
}

.exe-row:hover {
    border-color: rgba(34, 211, 238, 0.48);
    background: rgba(15, 23, 42, 0.75);
    transform: translateY(-1px);
}

.exe-os-pill {
    background: rgba(51, 65, 85, 0.85);
    border: 1px solid rgba(148, 163, 184, 0.3);
    border-radius: 999px;
    color: #cbd5e1;
    padding: 0.2rem 0.52rem;
    width: fit-content;
}

.exe-path-chip {
    text-align: center;
    border: 1px solid rgba(148, 163, 184, 0.3);
    border-radius: 8px;
    padding: 0.2rem 0.45rem;
    margin-right: 0.28rem;
    white-space: nowrap;
    color: #cbd5e1;
    background: rgba(15, 23, 42, 0.72);
}

.exe-action-button {
    color: #f8fafc;
    border-radius: 10px;
    padding: 0.34rem 0.72rem;
    border: 1px solid transparent;
    transition: transform var(--trans-fast), box-shadow var(--trans-fast), filter var(--trans-fast);
}

.exe-action-button:hover {
    transform: translateY(-1px);
    filter: brightness(1.04);
}

.exe-action-button-play {
    border-color: rgba(34, 211, 238, 0.45);
    background: linear-gradient(120deg, rgba(59, 130, 246, 0.92), rgba(34, 211, 238, 0.85));
    box-shadow: 0 0 16px rgba(34, 211, 238, 0.25);
}

.exe-action-button-stop {
    border-color: rgba(248, 113, 113, 0.5);
    background: linear-gradient(120deg, rgba(239, 68, 68, 0.93), rgba(248, 113, 113, 0.78));
    box-shadow: 0 0 16px rgba(239, 68, 68, 0.24);
}

.fade-right {
    -webkit-mask-image: linear-gradient(to right, black 85%, transparent 100%);
    mask-image: linear-gradient(to right, black 85%, transparent 100%);
}

.scrollbar-none {
    scrollbar-width: none;
    -ms-overflow-style: none;
}

.scrollbar-none::-webkit-scrollbar {
    display: none;
}
</style>
