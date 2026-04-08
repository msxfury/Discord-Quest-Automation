<script setup lang="ts">
import { ref, computed, useTemplateRef, shallowRef, provide, nextTick, triggerRef } from 'vue';
// import gameListData from '../assets/gamelist.json';
import { onClickOutside, refDebounced, tryOnMounted } from '@vueuse/core';
import { useFuse } from '@vueuse/integrations/useFuse'
import { invoke } from '@tauri-apps/api/core';
import { randomString } from '@/utils/random-string';
import { GameActionsProvider, GameExecutable, type Game } from '@/types/types';
import IconVerified from '@/components/IconVerified.vue';
import { isEmpty } from 'lodash-es';
import GameExecutables from '@/components/GameExecutables.vue';
import { GameActionsKey } from '@/constants/constants';
import { path } from '@tauri-apps/api';
import { emit } from '@tauri-apps/api/event';
import { useFetchGameList } from '@/composables/fetch-gamelist';
import { UseFuseOptions } from '@vueuse/integrations';
import Fuse from 'fuse.js';
import { useGlobalState } from '@/composables/app-state';
import TimedNotification from '@/components/TimedNotification.vue';


type DialogKey = 
    'none' | 
    'rpc_message_1'|
    'no_game_selected';;

// Game list from JSON file
// const gameDB = ref<Game[]>([]);

const {
    gameDB,
    isLoadingBundled,
    isLoadingDiscord,
    isLoadingGH,
    fetchGameList,
    isReadyGH,
    isReadyBundled,
    isReadyDiscord,
    allFetchDone,
} = useFetchGameList()
const { addLog } = useGlobalState();
const shouldShowNotificationContainer = computed(() => {
    return isLoadingGH.value || isLoadingDiscord.value || isLoadingBundled.value ||
           (isReadyGH.value || isReadyDiscord.value || isReadyBundled.value);
});

const dialogRef = useTemplateRef<HTMLDialogElement>('dialogRef');
const searchResultContainerRef = useTemplateRef<HTMLElement>('searchResultContainerRef')
const dialogMessage = ref('');
const isDialogOpen = ref(false);
const dialogKey = ref<DialogKey>('none')
const isConnectedToRPC = ref(false);
const isConnecting = ref(false);

// Search functionality
const searchQuery = shallowRef('');
const debouncedSearchQuery = refDebounced(searchQuery, 300)

const searchResultsIsOpen = ref(false);
const isOnSearchResults = ref(false);

// Game status
const currentlyPlaying = ref<string | null>(null);


onClickOutside(searchResultContainerRef, () => {
    searchResultsIsOpen.value = false;
})

// const searchResults = computed(() => {
//     if (!debouncedSearchQuery.value) return [];
//     const query = debouncedSearchQuery.value.toLowerCase();
//     return gameDB.value.filter(game =>
//         game.name.toLowerCase().includes(query) ||
//         game.aliases?.some(alias => alias.toLowerCase().includes(query))
//     );
// });

const COPYRIGHT_SYMBOL = '\u00A9';
const TRADEMARK_SYMBOL = '\u2122';
const REGISTERED_SYMBOL = '\u00AE';
const ignoredSymbols = [COPYRIGHT_SYMBOL, TRADEMARK_SYMBOL, REGISTERED_SYMBOL];
const ignoredSymbolsRegex = new RegExp(`[${ignoredSymbols.join('')}]`, 'g');
const fuseOptions = computed<UseFuseOptions<Game>>(() => ({
    fuseOptions: {
        // Prioritize name and aliases for searching, then lastly executables
        keys: [
            { name: 'name', weight: 0.7 },
            { name: 'aliases', weight: 0.2 },
            { name: 'executables.name', weight: 0.1 },
        ],
        getFn: (obj: any, path: string[] | string) => {
            const value = Fuse.config.getFn(obj, path);
            return typeof value === "string"
            ? value.replace(ignoredSymbolsRegex, "")
            : value;
        },
        isCaseSensitive: false,
        threshold: 0.5,        
        // A score of 0indicates a perfect match, while a score of 1 indicates a complete mismatch
        includeScore: true,
        includeMatches: false
    },
    resultLimit: 12,
    matchAllWhenSearchEmpty: false,
}));

const { results: searchResults } = useFuse(debouncedSearchQuery, gameDB, fuseOptions)

// Selected games list
const gameList = ref<Game[]>([]);
// const selectedGame = ref<Game | null>(null);
const selectedGameId = ref<string | null | undefined>(null);

const selectedGame = computed(() => {
    if (!selectedGameId.value) return null;
    const found = gameList.value.find(g => g.uid === selectedGameId.value);
    console.log('selectedGame computed - selectedGameId:', selectedGameId.value, 'found:', found);
    return found || null;
});

function closeSearchResults() {
    searchResultsIsOpen.value = false;
}
function openSearchResults() {
    searchResultsIsOpen.value = true;
}

// Function to add a game to the selected list
function addGameToList(game: Game) {
    if (!gameList.value.some(g => g.id === game.id)) {
        gameList.value.push({
            uid: randomString(),
            ...game
        });
    }

    closeSearchResults();
}

const forceRerenderKey = ref(0); 
// Function to remove a game from the selected list
function removeGameFromList(game: Game) {
    const gameId = game.uid;
    gameList.value = gameList.value.filter(game => game.uid !== gameId);
    if (selectedGame.value?.uid === gameId) { 
        // selectedGame.value = null;
        selectedGameId.value = null;
        forceRerenderKey.value++; 
    }
}

function selectGame(game: Game) {
    // selectedGame.value = game;
    selectedGameId.value = game?.uid;
    searchResultsIsOpen.value = false;
}

function canCreateDummyGame(game: Game | null) {
    if (!game) {
        return false;
    }
    // we can only create a dummy game if the game is not installed or game is not running
    return !game.is_installed
}

function canPlayGame(game: Game | null) {
    if (!game) {
        return false;
    }
    // we can only play a game if the game is installed and not running
    return (game.is_installed && !game.is_running) ?? false;
}

function isExecutableRunning(executable: GameExecutable) {
    // Check if the executable is running
    return executable.is_running ?? false;
}
function isGameExecutableInstalled(executable: GameExecutable) {
    // Check if the executable is installed
    return executable.is_installed ?? false;
}

function isGameInstalled(game: Game | null) {
    if (!game) {
        return false;
    }
    // we can only play a game if the game is installed and not running
    return game.is_installed ?? false;
}


// Create a dummy game
async function createDummyGame(game: Game | null, executable: GameExecutable) {
    if (!game) {
        return;
    }
    const gameUid = game.uid;
    const gameToInstall = gameList.value.find(g => g.uid === gameUid);
    const executableItem = gameToInstall?.executables.find(exe => exe.name === executable.name);
    if (gameToInstall && executableItem) {
        const payload =  { 
            path: executable.path,
            executable_name: executable.filename,
            path_len: executable.segments,
            app_id: Number(gameToInstall.id),
        }
        console.log(payload);
        const result = await invoke('create_fake_game', payload)
        console.log('Game created:', result);
        gameToInstall.is_installed = true;
        executableItem.is_installed = true;
        return true;
    }
}


async function installAndPlay({game, executable}: {game: Game, executable: GameExecutable}) {
    if (!game) {
        return;
    }
    const gameCreated = await createDummyGame(game, executable);
    if (gameCreated) {
        playGame({game, executable});
    } else {
        console.error('Failed to create game');
        addLog('error', 'Failed to create game');
    }
}
// Play game function
async function playGame({game, executable}: {game: Game, executable: GameExecutable}) {
    if (!game) {
        return;
    }
    const gameUid = game.uid;
    try {
        console.log(`Playing game: ${gameUid}`);
        addLog('info', `Playing game: ${game.name}`);
        addLog('info', `Executable: ${executable.name}`);
        currentlyPlaying.value = game.id;
        // find the game in the list
        const gameToPlay = gameList.value.find(g => g.uid === gameUid);
        const executableItem = gameToPlay?.executables.find(exe => exe.name === executable.name);
        if (gameToPlay && executableItem) {
            const payload =  { 
                name: game.name,
                path: executable.path,
                executable_name: executable.filename,
                path_len: executable.segments,
                app_id: Number(gameToPlay.id),
                exec_path: path.join(executable.path!, executable.filename!),
            } 
            await invoke('run_background_process', payload);
            gameToPlay.is_running = true;
            executableItem.is_running = true; 
        }
        // In a real app, this would invoke a Tauri command to launch the game
       
    } catch (error) {
        console.error('Failed to launch game:', error);
    }
}

// Stop playing
async function stopPlaying({game, executable}: {game: Game, executable: GameExecutable}) {
    if (!game) {
        return;
    }
    console.log('Stopped playing game');
    const gameUid = game.uid;
    
    currentlyPlaying.value = null;

    const gameToPlay = gameList.value.find(g => g.uid === gameUid);
    const executableItem = gameToPlay?.executables.find(exe => exe.name === executable.name);
    if (gameToPlay && executableItem) {
        try {
            await invoke('stop_process', {
                exec_name: executable.filename!
            })
            addLog('info', `Stopped game process: ${game.name}`);
            addLog('info', `Stopped Executable: ${executable.name}`);
        } catch (error) {
            console.error('Failed to stop game process:', error);
            const errorMessage = (error instanceof Error) ? error.message : String(error);
            addLog('error', 'Failed to stop game process' + errorMessage);
            // Even if stopping fails, we still update the state
            gameToPlay.is_running = false;
            executableItem.is_running = false;
        } finally {
            gameToPlay.is_running = false;
            executableItem.is_running = false;
        }
    }
}

function getExecutables(game: Game) {
    return game.executables.map(exe => exe.name)
}

async function handleTestRPC(game: Game | null) {
    let state = isConnectedToRPC.value ? 'disconnect' : 'connect';

    console.log('Testing RPC for game:', game);
    if (!game && state === 'connect') {
        showDialog('no_game_selected');
        return;
    }
    if (state === 'disconnect' || isConnecting.value) {
        // await invoke('connect_to_discord_rpc_2', { app_id: "0", discord_state: "disconnect" })
        // invoke('connect_to_discord_rpc_3', {
        //     activity_json: JSON.stringify({
        //         app_id: selectedGame.value?.id
        //     }),
        //     action: 'disconnect',
        // })
        emit('event_disconnect');
        
        isConnectedToRPC.value = false;
        game!.is_running = false;
        currentlyPlaying.value = null;
        isConnecting.value = false;
        return;
    }
    showDialog('rpc_message_1');
}

async function continueRPCRisk(game: Game | null) {
    if (!game) {
        return;
    }
    const gameUid = game.uid;
    const gameToTest = gameList.value.find(g => g.uid === gameUid);
    if (gameToTest) {
        console.log('Testing RPC for game:', gameToTest);
        isConnecting.value = true;
        // invoke('connect_to_discord_rpc_2', { app_id: gameToTest.id, discord_state: "connect" })
        invoke('connect_to_discord_rpc_3', {
            activity_json: JSON.stringify({
                app_id: gameToTest.id,
            }),
            action: 'connect',
        })
        .then(() => {
            isConnectedToRPC.value = true;
            gameToTest.is_running = true;
            currentlyPlaying.value = gameToTest.id;
            isConnecting.value = false;
        })

        hideDialog();
    }
}

function handleSearchBlur() {
    setTimeout(() => {
        if (!isOnSearchResults.value) {
            searchResultsIsOpen.value = false;
        }
    }, 200);
}

function showDialog(message: DialogKey) {
    isDialogOpen.value = true;
    dialogMessage.value = message;
    dialogKey.value = message;
    if(!isEmpty(message)) {
        dialogRef.value?.showModal();
    }
}

function hideDialog() {
    dialogRef.value?.close(); 
    dialogMessage.value = '';
    isDialogOpen.value = false;
}


provide<GameActionsProvider>(GameActionsKey, {
    canPlayGame,
    isGameInstalled,
    isExecutableRunning,
    isGameExecutableInstalled,
});
</script>

<template>
    <div class="home-root container mx-auto px-4 py-8 md:px-8">
        <dialog
            id="dialog"
            class="dialogStyle"
            style="left: 50%; top: 50%; transform: translate(-50%, -50%)"
            ref="dialogRef"
        >
            <div class="flex flex-col items-center justify-center p-6 md:p-7">
                <div class="mb-4 text-center text-slate-300 text-sm leading-relaxed">
                    <div v-if="dialogKey === 'rpc_message_1'">
                        <p>This feature is currently in development.</p>
                        <p class="my-2">
                            It may trick Discord into thinking a real game is running by sending RPC with an actual game ID.
                        </p>
                        <p>This could flag your account as suspicious for self-botting.</p>
                    </div>
                    <div v-if="dialogKey === 'no_game_selected'">
                        <p>No game selected. Please select a game from the Games panel first.</p>
                    </div>
                </div>
                <div class="gap-2 flex">
                    <button class="dialog-button" @click="hideDialog()">
                        <span v-if="dialogKey == 'rpc_message_1'">Cancel</span>
                        <span v-else>OK</span>
                    </button>
                    <button
                        v-if="dialogKey === 'rpc_message_1'"
                        class="dialog-button dialog-button-primary"
                        @click="continueRPCRisk(selectedGame)"
                    >
                        Accept Risk and Continue
                    </button>
                </div>
            </div>
        </dialog>

        <section class="hero-panel">
            <div>
                <p class="hero-kicker">Realtime Command Center</p>
                <h1 class="hero-title">Discord Quest Handler</h1>
                <p class="hero-subtitle">
                    Manage verified games, trigger actions, and monitor runtime status from a focused, high-clarity dashboard.
                </p>
            </div>
            <div class="hero-chip">
                <span
                    class="status-dot"
                    :class="{
                        'status-online': currentlyPlaying,
                        'status-connected': !currentlyPlaying && isConnectedToRPC,
                        'status-idle': !currentlyPlaying && !isConnectedToRPC
                    }"
                ></span>
                {{ currentlyPlaying ? 'Running' : (isConnectedToRPC ? 'Connected' : 'Idle') }}
            </div>
        </section>

        <Transition
            enter-active-class="transition-opacity duration-300 delay-100 ease-in-out"
            leave-active-class="transition-opacity duration-500 delay-100 ease-in-out"
            enter-from-class="opacity-0 translate-y-2"
            enter-to-class="opacity-100 translate-y-0"
        >
            <div class="notify-stack" v-if="shouldShowNotificationContainer && !allFetchDone">
                <Transition
                    enter-active-class="transition-opacity duration-300 ease-in-out"
                    leave-active-class="transition-opacity duration-400 ease-in-out"
                    enter-from-class="opacity-0 translate-y-2"
                    enter-to-class="opacity-100 translate-y-0"
                >
                    <div v-if="isLoadingGH" class="notify-pill">
                        Fetching game list from GitHub mirror...
                        <span class="notify-dot animate-pulse"></span>
                    </div>
                </Transition>
                <TimedNotification :is-ready="isReadyGH" :duration="1500" container-class="notify-pill notify-pill-success">
                    Game list from mirror fetched
                </TimedNotification>

                <Transition
                    enter-active-class="transition-opacity duration-300 ease-in-out"
                    leave-active-class="transition-opacity duration-400 ease-in-out"
                    enter-from-class="opacity-0 translate-y-2"
                    enter-to-class="opacity-100 translate-y-0"
                >
                    <div v-if="isLoadingDiscord" class="notify-pill">
                        Fetching game list directly from Discord...
                        <span class="notify-dot animate-pulse"></span>
                    </div>
                </Transition>
                <TimedNotification :is-ready="isReadyDiscord" :duration="1500" container-class="notify-pill notify-pill-success">
                    Game list from Discord fetched
                </TimedNotification>

                <Transition
                    enter-active-class="transition-opacity duration-300 ease-in-out"
                    leave-active-class="transition-opacity duration-400 ease-in-out"
                    enter-from-class="opacity-0 translate-y-2"
                    enter-to-class="opacity-100 translate-y-0"
                >
                    <div v-if="isLoadingBundled" class="notify-pill">
                        Fetching game list from bundled game list...
                        <span class="notify-dot animate-pulse"></span>
                    </div>
                </Transition>
                <TimedNotification :is-ready="isReadyBundled" :duration="1500" container-class="notify-pill notify-pill-success">
                    Game list from bundle pre-loaded
                </TimedNotification>
            </div>
        </Transition>

        <section class="search-panel mb-8">
            <div class="relative" ref="searchResultContainerRef">
                <div class="search-bar-wrap">
                    <input
                        v-model="searchQuery"
                        type="text"
                        placeholder="Search Discord verified games..."
                        class="search-input"
                        @focus="openSearchResults"
                        @blur="handleSearchBlur"
                    />
                    <button @click="fetchGameList()" class="secondary-button">
                        <span class="whitespace-nowrap text-xs md:text-sm">Refetch Game List</span>
                    </button>
                </div>
                <div v-if="searchResultsIsOpen" @click="isOnSearchResults = true" class="search-results">
                    <div v-if="searchResults.length > 0">
                        <div v-for="game in searchResults" :key="game.item.id" class="search-result-row">
                            <div class="flex justify-between items-start gap-4">
                                <div>
                                    <div class="font-medium text-slate-100">{{ game.item.name }}</div>
                                    <div class="text-sm text-slate-400">ID: {{ game.item.id }}</div>
                                    <div class="text-xs text-slate-400 mt-1.5">
                                        Executables:
                                        <ul class="list-disc list-inside">
                                            <li v-for="exe in game.item.executables" :key="exe.name">
                                                <span class="font-mono">{{ exe.name }} ({{ exe.os }})</span>
                                            </li>
                                        </ul>
                                    </div>
                                </div>
                                <button @click="addGameToList(game.item)" class="mini-primary-button">Add Game</button>
                            </div>
                        </div>
                    </div>
                    <div v-if="searchResults.length === 0" class="p-4 text-sm text-slate-400">
                        Search by game name and click <span class="text-slate-200">Add Game</span> to include it in your active list.
                    </div>
                </div>
            </div>
        </section>

        <section class="grid grid-cols-1 xl:grid-cols-[1.15fr_0.85fr] gap-6 relative">
            <div class="panel-card">
                <h2 class="panel-title">Games</h2>
                <div v-if="gameList.length === 0" class="empty-state">
                    <div class="empty-icon-wrap">
                        <svg viewBox="0 0 24 24" fill="none" class="h-8 w-8 text-cyan-300">
                            <path d="M5 9.5h14a2 2 0 0 1 2 2v4a2 2 0 0 1-2 2h-2l-2-2h-6l-2 2H5a2 2 0 0 1-2-2v-4a2 2 0 0 1 2-2Z" stroke="currentColor" stroke-width="1.5" />
                            <path d="M8 7V5m8 2V5m-8 7h2m4 0h2" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" />
                        </svg>
                    </div>
                    <h3 class="text-lg font-semibold text-slate-100">No games selected</h3>
                    <p class="text-sm text-slate-400 max-w-md">
                        Start by searching for a Discord-verified game, then add it to your active games list to unlock actions.
                    </p>
                    <button class="mini-primary-button mt-1" @click="openSearchResults()">Add Game</button>
                </div>
                <div v-else class="space-y-3">
                    <div
                        v-for="game in gameList"
                        :key="game.id"
                        class="game-row"
                        :class="[
                            {
                                'game-row-active': selectedGame?.uid === game.uid,
                            }
                        ]"
                        @click="selectGame(game)"
                    >
                        <div class="flex justify-between items-center gap-3">
                            <div class="flex items-center gap-2">
                                <div class="font-medium text-slate-100">{{ game.name }}</div>
                                <div class="relative inline-flex items-center">
                                    <div class="w-2 h-2 bg-white absolute rounded-full" style="left: 50%; top: 50%; transform: translate(-50%, -50%)"></div>
                                    <div class="relative inline-block">
                                        <IconVerified class="w-5 h-5 text-violet-400"></IconVerified>
                                    </div>
                                </div>
                            </div>
                            <button @click="removeGameFromList(game)" class="remove-button" v-if="!game.is_running">Remove</button>
                        </div>
                        <div class="mt-2 text-sm text-emerald-300" v-if="game.is_running">Running</div>
                    </div>
                </div>
            </div>

            <div class="panel-card md:sticky md:top-4 self-start" :key="forceRerenderKey">
                <h2 class="panel-title">Game Actions</h2>
                <div class="space-y-4">
                    <div class="text-slate-400 mb-2 text-sm" v-if="!selectedGame || selectedGame === null">
                        Select a game from the Games panel to perform actions.
                    </div>
                    <div v-if="selectedGame" class="text-slate-300 mb-4 text-sm leading-6">
                        <strong class="text-slate-100">Name:</strong> {{ selectedGame.name }}<br>
                        <strong class="text-slate-100">ID:</strong> {{ selectedGame.id }}<br>
                        <strong v-if="selectedGame.aliases && selectedGame.aliases.length > 0" class="text-slate-100">Aliases:</strong>
                        <ul v-if="selectedGame.aliases && selectedGame.aliases.length > 0" class="list-disc list-inside">
                            <li v-for="alias in selectedGame.aliases" :key="alias">
                                <span class="font-mono text-slate-300">{{ alias }}</span>
                            </li>
                        </ul>
                    </div>
                    <button @click="handleTestRPC(selectedGame)" class="primary-button">
                        {{ isConnecting || isConnectedToRPC ? 'Disconnect from Discord Gateway' : 'Test RPC' }}
                    </button>

                    <div class="panel-divider"></div>

                    <GameExecutables
                        v-if="selectedGame"
                        :game="selectedGame"
                        @play="playGame"
                        @stop="stopPlaying"
                        @install_and_play="installAndPlay"
                    />
                </div>

                <div class="panel-divider my-5"></div>

                <div class="status-card">
                    <h3 class="font-semibold text-slate-100 mb-2 text-base">Status</h3>
                    <div class="text-sm text-slate-400 mb-3">
                        Check Discord to verify if activity appears as expected.
                    </div>
                    <div class="status-row">
                        <span
                            class="status-dot"
                            :class="{
                                'status-online': currentlyPlaying,
                                'status-connected': !currentlyPlaying && isConnectedToRPC,
                                'status-idle': !currentlyPlaying && !isConnectedToRPC
                            }"
                        ></span>
                        <span class="text-sm text-slate-300">
                            {{ currentlyPlaying ? 'Running' : (isConnectedToRPC ? 'Connected' : 'Idle') }}
                        </span>
                    </div>
                    <div class="status-row mt-2" v-if="currentlyPlaying">
                        <span class="text-sm text-slate-300">
                            Ready: {{ gameList.find(g => g.id === currentlyPlaying)?.name }}
                        </span>
                    </div>
                    <div class="status-row mt-2" v-else>
                        <span class="text-sm text-slate-400">Ready: No active game</span>
                    </div>
                </div>

                <div v-if="selectedGame" class="my-4">
                    <h3 class="font-medium text-slate-100 mb-2">Game Info</h3>
                </div>
            </div>
        </section>
    </div>
</template>

<style scoped>
@reference "../theme/style.css";

.home-root {
    position: relative;
}

.hero-panel {
    border: 1px solid rgba(148, 163, 184, 0.2);
    border-radius: var(--radius-lg);
    background:
        linear-gradient(130deg, rgba(124, 58, 237, 0.2), rgba(34, 211, 238, 0.1)),
        var(--bg-surface);
    box-shadow: var(--shadow-soft);
    backdrop-filter: blur(10px);
    padding: 1.25rem 1.25rem;
    margin-bottom: 1rem;
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    justify-content: space-between;
    gap: 1rem;
}

.hero-kicker {
    color: #a5b4fc;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    font-size: 0.72rem;
    margin-bottom: 0.35rem;
}

.hero-title {
    color: #f8fafc;
    font-size: clamp(1.5rem, 2.5vw, 2.1rem);
    line-height: 1.2;
    font-weight: 700;
}

.hero-subtitle {
    margin-top: 0.45rem;
    color: #94a3b8;
    max-width: 740px;
    font-size: 0.95rem;
}

.hero-chip {
    border: 1px solid rgba(148, 163, 184, 0.3);
    border-radius: 999px;
    padding: 0.45rem 0.75rem;
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    background: rgba(15, 23, 42, 0.6);
    color: #cbd5e1;
    font-size: 0.85rem;
}

.dialogStyle::backdrop {
    @apply bg-black/75 backdrop-blur-xs;
}

.dialogStyle {
    min-width: min(92vw, 640px);
    border-radius: var(--radius-lg);
    border: 1px solid rgba(148, 163, 184, 0.25);
    background: linear-gradient(180deg, rgba(19, 30, 52, 0.97), rgba(10, 17, 31, 0.95));
    box-shadow: var(--shadow-soft), var(--shadow-glow);
}

.dialog-button {
    border-radius: 999px;
    border: 1px solid rgba(148, 163, 184, 0.35);
    color: #cbd5e1;
    background: rgba(15, 23, 42, 0.65);
    padding: 0.5rem 1rem;
    transition: all var(--trans-fast);
}

.dialog-button:hover {
    color: #f8fafc;
    border-color: rgba(148, 163, 184, 0.6);
    transform: translateY(-1px);
}

.dialog-button-primary {
    color: #f8fafc;
    border-color: rgba(124, 58, 237, 0.45);
    background: linear-gradient(120deg, rgba(124, 58, 237, 0.9), rgba(34, 211, 238, 0.8));
}

.notify-stack {
    position: fixed;
    top: 5.8rem;
    right: 1rem;
    z-index: 30;
    display: flex;
    flex-direction: column;
    gap: 0.45rem;
}

.notify-pill {
    border-radius: 999px;
    border: 1px solid rgba(148, 163, 184, 0.26);
    color: #cbd5e1;
    background: rgba(15, 23, 42, 0.85);
    font-size: 0.78rem;
    padding: 0.42rem 0.75rem;
    display: inline-flex;
    align-items: center;
    gap: 0.45rem;
}

.notify-pill-success {
    border-color: rgba(34, 197, 94, 0.45);
    color: #bbf7d0;
}

.notify-dot {
    width: 0.5rem;
    height: 0.5rem;
    border-radius: 999px;
    background: #22c55e;
}

.search-panel {
    position: relative;
    z-index: 20;
}

.search-bar-wrap {
    border: 1px solid rgba(148, 163, 184, 0.24);
    background: rgba(12, 19, 34, 0.85);
    border-radius: var(--radius-lg);
    padding: 0.45rem;
    display: grid;
    grid-template-columns: 1fr auto;
    gap: 0.6rem;
    box-shadow: var(--shadow-soft);
    transition: border-color var(--trans-fast), box-shadow var(--trans-base), transform var(--trans-fast);
}

.search-bar-wrap:focus-within {
    border-color: rgba(124, 58, 237, 0.55);
    box-shadow: var(--shadow-soft), 0 0 24px rgba(124, 58, 237, 0.35);
    transform: translateY(-1px);
}

.search-input {
    width: 100%;
    border-radius: 12px;
    border: 1px solid rgba(148, 163, 184, 0.25);
    background: rgba(16, 26, 46, 0.72);
    color: #f8fafc;
    font-size: 0.95rem;
    padding: 0.7rem 0.9rem;
    outline: none;
    transition: border-color var(--trans-fast), background var(--trans-fast), box-shadow var(--trans-fast);
}

.search-input::placeholder {
    color: #64748b;
}

.search-input:focus {
    border-color: rgba(34, 211, 238, 0.6);
    background: rgba(12, 19, 34, 0.92);
}

.secondary-button {
    border: 1px solid rgba(148, 163, 184, 0.35);
    color: #dbeafe;
    background: rgba(30, 41, 59, 0.8);
    border-radius: 12px;
    padding: 0.6rem 0.9rem;
    transition: all var(--trans-fast);
}

.secondary-button:hover {
    color: #ffffff;
    border-color: rgba(34, 211, 238, 0.5);
    background: rgba(30, 41, 59, 1);
}

.search-results {
    position: absolute;
    z-index: 60;
    margin-top: 0.55rem;
    width: 100%;
    max-height: 21rem;
    overflow-y: auto;
    border-radius: var(--radius-lg);
    border: 1px solid rgba(148, 163, 184, 0.25);
    background: rgba(11, 18, 32, 0.96);
    backdrop-filter: blur(8px);
    box-shadow: var(--shadow-soft);
}

.search-result-row {
    padding: 0.85rem 0.95rem;
    border-bottom: 1px solid rgba(148, 163, 184, 0.16);
    transition: background-color var(--trans-fast);
}

.search-result-row:last-child {
    border-bottom: none;
}

.search-result-row:hover {
    background: rgba(30, 41, 59, 0.5);
}

.panel-card {
    border: 1px solid rgba(148, 163, 184, 0.2);
    border-radius: var(--radius-lg);
    background:
        linear-gradient(160deg, rgba(124, 58, 237, 0.08), transparent 45%),
        rgba(13, 21, 38, 0.88);
    box-shadow: var(--shadow-soft);
    backdrop-filter: blur(10px);
    padding: 1.1rem;
    transition: transform var(--trans-fast), border-color var(--trans-fast), box-shadow var(--trans-base);
}

.panel-card:hover {
    transform: translateY(-2px);
    border-color: rgba(124, 58, 237, 0.35);
    box-shadow: var(--shadow-soft), 0 0 26px rgba(124, 58, 237, 0.22);
}

.panel-title {
    font-size: 1.2rem;
    font-weight: 700;
    color: #f8fafc;
    margin-bottom: 0.9rem;
}

.empty-state {
    min-height: 320px;
    border: 1px dashed rgba(148, 163, 184, 0.28);
    border-radius: 14px;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    text-align: center;
    gap: 0.65rem;
    padding: 1.2rem;
}

.empty-icon-wrap {
    width: 3.2rem;
    height: 3.2rem;
    border-radius: 999px;
    border: 1px solid rgba(34, 211, 238, 0.4);
    display: inline-flex;
    align-items: center;
    justify-content: center;
    background: rgba(34, 211, 238, 0.12);
}

.game-row {
    border: 1px solid rgba(148, 163, 184, 0.2);
    border-radius: 14px;
    padding: 0.8rem 0.9rem;
    background: rgba(15, 23, 42, 0.62);
    cursor: pointer;
    transition: all var(--trans-fast);
}

.game-row:hover {
    border-color: rgba(34, 211, 238, 0.45);
    transform: translateY(-1px);
}

.game-row-active {
    border-color: rgba(124, 58, 237, 0.6);
    box-shadow: 0 0 22px rgba(124, 58, 237, 0.22);
    background: rgba(76, 29, 149, 0.23);
}

.remove-button {
    color: #fca5a5;
    font-size: 0.8rem;
    border: 1px solid rgba(252, 165, 165, 0.35);
    border-radius: 999px;
    padding: 0.2rem 0.6rem;
    transition: all var(--trans-fast);
}

.remove-button:hover {
    color: #fecaca;
    border-color: rgba(252, 165, 165, 0.65);
    background: rgba(127, 29, 29, 0.35);
}

.primary-button {
    width: 100%;
    border-radius: 12px;
    border: 1px solid rgba(124, 58, 237, 0.45);
    color: #f8fafc;
    font-weight: 600;
    font-size: 0.95rem;
    padding: 0.72rem 1rem;
    background: linear-gradient(110deg, #7c3aed 0%, #22d3ee 100%);
    box-shadow: 0 0 24px rgba(124, 58, 237, 0.35);
    transition: transform var(--trans-fast), box-shadow var(--trans-fast), filter var(--trans-fast);
}

.primary-button:hover {
    transform: translateY(-1px) scale(1.01);
    box-shadow: 0 0 28px rgba(34, 211, 238, 0.4);
    filter: brightness(1.03);
}

.mini-primary-button {
    border-radius: 999px;
    border: 1px solid rgba(124, 58, 237, 0.45);
    color: #f8fafc;
    font-size: 0.8rem;
    font-weight: 600;
    padding: 0.42rem 0.72rem;
    background: linear-gradient(120deg, rgba(124, 58, 237, 0.9), rgba(34, 211, 238, 0.88));
    transition: transform var(--trans-fast), box-shadow var(--trans-fast);
}

.mini-primary-button:hover {
    transform: translateY(-1px);
    box-shadow: 0 0 18px rgba(124, 58, 237, 0.35);
}

.panel-divider {
    border-top: 1px solid rgba(148, 163, 184, 0.2);
}

.status-card {
    border: 1px solid rgba(148, 163, 184, 0.22);
    border-radius: 14px;
    background: rgba(10, 17, 31, 0.78);
    padding: 0.95rem;
}

.status-row {
    display: flex;
    align-items: center;
    gap: 0.45rem;
}

.status-dot {
    width: 0.6rem;
    height: 0.6rem;
    border-radius: 999px;
    box-shadow: 0 0 10px currentColor;
}

.status-online {
    color: #22c55e;
    background: #22c55e;
}

.status-idle {
    color: #f59e0b;
    background: #f59e0b;
}

.status-connected {
    color: #22d3ee;
    background: #22d3ee;
}

@media (max-width: 768px) {
    .notify-stack {
        position: relative;
        top: 0;
        right: 0;
        margin-bottom: 0.8rem;
    }

    .search-bar-wrap {
        grid-template-columns: 1fr;
    }
}
</style>
