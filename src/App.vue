<template>
    <div v-if="currentPage === 'before'">
        <div class="bg-white p-5 rounded-lg shadow-md w-[85%] max-w-[400px] m-auto absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 text-black">

          <div v-if="!roomOption">
            <button @click="nextRoomOption('create')" class="bg-[#3581B8] text-white  block w-1/2 mx-auto p-[10px] rounded">Create a room</button>
            <hr class="my-3">
            <button @click="nextRoomOption('join'); " class="bg-[#13563B] text-white block w-1/2 mx-auto p-[10px] rounded">Join a room</button>
          </div>
          <template v-if="roomOption && !roomCode">
            <h2>Type your name</h2>
              <div class="flex items-center mb-3 border border-gray-300 rounded overflow-hidden">
              <input
                type="text"
                class="flex-grow p-2 outline-none"
                v-model="username"
                placeholder="Enter your username"
              >
              <button
                class="p-2 text-gray-600 hover:text-gray-900"
                @click="username = getRandomName()"
              >
                <i class="fa-solid fa-shuffle"></i>
              </button>
            </div>

              <div class="w-[95%] mx-auto grid grid-cols-5 gap-3 justify-between mb-5">
                <div v-for="(avatar, index) in avatars" :key="index" @click="randomString = avatar.randomString" v-html="avatar.avatar"  :style="{ opacity:  randomString !== avatar.randomString ? '0.6' : '1' }"></div>

                <div class="text-3xl text-gray-600 flex justify-center items-center text-center" @click="generateAvatars('female')">
                  <i class="fas fa-sync"></i>
                </div>
              </div>
              <div class="flex items-center mb-2" v-if="roomOption === 'join'">
                <input type="number" v-model="tempRoomcode" placeholder="Type room code" class="flex-grow p-2 border border-gray-300 rounded">
              </div>
              <!-- <button v-if="readyToPlay" @click="randomName()" class="add-button">ランダム</button> -->
              <button @click="roomOption = null" class="bg-[#B83A4B] text-white block mx-auto p-[10px] rounded w-1/2 mb-2">Back</button>
              <button @click="username = getRandomName();" class="bg-[black] text-white block mx-auto p-[10px] rounded mb-2 w-1/2">Random Name</button>
              <button v-if="readyToPlay && roomOption === 'create'" @click="createARoom()" class="bg-[#3581B8] text-white  block  mx-auto p-[10px] rounded w-1/2 mb-2">Create</button>
              <button v-if="tempRoomcode >= 10000 && tempRoomcode <= 99999 && readyToPlay && roomOption === 'join'" @click="joinARoom()" class="bg-[#3581B8] text-white  block mx-auto p-[10px] rounded w-1/2 mb-2">Join</button>
              <!-- <button v-if="tempRoomcode >= 10000 && tempRoomcode <= 99999 && readyToPlay && roomOption === 'join'" @click="monitorGame()" class="bg-yellow-400 text-white  block mx-auto p-[10px] rounded w-1/2">Monitor mode</button> -->
          </template>

          <template v-if="roomOption && roomCode">
            <template  v-if="isHost"><h2>You are host</h2></template>
              
            <h2 v-if="!isHost">Welcome {{ username }}!</h2>
            <p>Room code: <strong class="font-size: 2.5em; color: crimson; margin-right: 5px; font-weight: bold;">{{ roomCode }}</strong></p>
            <hr>
            <template v-for="(player, index) in players" :key="index">
              <div class="player-list flex items-center gap-2 my-2">
                <span>{{index +1}}.</span>
                <div class="block w-10 aspect-square" v-html="regenerate(player?.randomString)"></div>
                <p>{{ player.name }}</p>
              </div>
            </template>
            <button v-if="players?.length == 2 && isHost"  @click="closeTheRoom()" class="bg-[#3581B8] text-white  block  mx-auto p-[10px] rounded w-1/2 mb-2">Close room</button>
          </template>
        </div>
    </div>
    <div v-if="currentPage === 'game'" class="p-4 flex flex-col items-center float-right">

        <div class="grid grid-cols-2 gap-4 p-4 bg-white/60 backdrop-blur-md border border-gray-300 rounded-xl shadow-md">
            <!-- <button
                @click="undoMove"
                class="px-4 py-2 bg-indigo-500 text-white rounded hover:bg-indigo-600 transition"
            >
                <i class="fa-solid fa-rotate-left"></i>
            </button> -->
            <button
                v-if="replayModeOn"
                @click="toggleAutoReplay"
                class="px-4 py-2 bg-indigo-500 text-white rounded hover:bg-indigo-600 transition"
            >
                <i
                    class="fa-solid"
                    :class="isAutoReplay ? 'fa-pause' : 'fa-play'"
                ></i>
            </button>
            <button
                v-if="replayModeOn"
                @click="nextMove"
                class="px-4 py-2 bg-indigo-500 text-white rounded hover:bg-indigo-600 transition"
            >
                <i class="fa-solid fa-rotate-right"></i>
            </button>

            <button
                    @click="resetBoard"
                    class="px-4 py-2 bg-red-500 text-white rounded  transition"
                >
                <i class="fa-solid fa-trash"></i>
                </button>

                <button
                    @click="replayModal = true"
                    class="px-4 py-2 bg-green-500 text-white rounded  transition"
                >
                <i class="fas fa-save"></i>
            </button>
            
        </div>


        <h3 v-if="winner" class="mt-4 font-xl font-bold text-white">Winner: {{ winner }}!</h3>
        <hr>

        <div :class="myPlayer.color == 'black' ? 'rotate-180' : ''" class="max-w-[725px] w-[92.5vw] aspect-square absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 border-4 border-gray-300 shadow-lg">
            <!-- Board squares -->
            <div class="grid grid-cols-8 grid-rows-8 w-full h-full">
                <div
                    v-for="square in boardSquares"
                    :key="`${square.row}-${square.col}`"
                    :class="[
                        squareClass(square.row, square.col),
                        isPossibleMove(square.row, square.col) ? 'blink' : '',
                        isLastMove(square.row, square.col) ? 'square--last-move' : ''
                    ]"
                    @click="moveToTile(square.row, square.col)"
                ></div>
            </div>

            <!-- Pieces layer -->
            <div class="absolute top-0 left-0 w-full h-full pointer-events-none">
                <div
                    v-for="piece in pieces"
                    :key="piece.id"
                    class="piece absolute text-[35px] md:text-[50px] transition-all duration-300 cursor-pointer pointer-events-auto"
                    :style="{
                        top: `${piece.row * 12.5}%`,
                        left: `${piece.col * 12.5}%`
                    }"
                    @click.stop="selectPiece(piece)"
                    :class="[
                        selected && selected.id === piece.id ? 'ring-4 ring-indigo-400' : '',
                        myPlayer.color === 'black' ? 'rotate-180' : ''
                    ]"
                >
                <i
                    :class="['fa-solid', `fa-chess-${piece.type}`]"
                    :style="{ color: piece.color === 'white' ? '#f8f8f8' : '#111' }"
                ></i>
                </div>
            </div>

        </div>

        <!-- Promotion Modal -->
        <div v-if="promotionModal" class="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
            <div class="bg-white rounded-xl p-6 w-80 text-center shadow-lg">
                <h2 class="text-lg font-semibold mb-4">Promote Pawn</h2>
                <p class="mb-4">Choose a piece:</p>
                <div class="flex justify-around gap-2">
                    <button 
                        v-for="type in ['queen','rook','bishop','knight']"
                        :key="type"
                        @click="promotePawn(type)"
                        class="flex-1 py-2 rounded-lg bg-indigo-500 text-white hover:bg-indigo-600 transition"
                    >
                        <i :class="`fa-solid fa-chess-${type}`"></i>
                        <br>
                        <span class="ml-1 capitalize">{{ type }}</span>
                    </button>
                </div>
            </div>
        </div>

        <!-- Input Modal -->
        <div v-if="replayModal" class="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
            <div class="bg-white rounded-xl p-6 w-80 text-center shadow-lg">
                <h2 class="text-lg font-semibold mb-4">Replay Game</h2>
                <p class="mb-4">Paste your UCI move log:</p>
                <textarea
                    v-model="replayLogInput"
                    class="w-full h-32 p-2 border border-gray-300 rounded mb-4 resize-none"
                    placeholder="e2e4,e7e5,g1f3,..."
                ></textarea>
                <div class="flex justify-end gap-2">
                    <button
                        @click="replayModal = false"
                        class="px-4 py-2 bg-gray-300 rounded hover:bg-gray-400 transition"
                    >Cancel</button>
                    <button
                        @click="startReplay"
                        class="px-4 py-2 bg-green-500 text-white rounded hover:bg-green-600 transition"
                    >Start Replay</button>
                </div>
            </div>
        </div>

        <div class="info-container">
            <div :class="getClassForPlayer(opponentPlayer.color)" class="top-16">
                <p>{{ opponentPlayer?.name }}</p>
                <div class="block w-10 aspect-square my-2 mx-auto" v-html="regenerate(opponentPlayer?.randomString)"></div>
                <p class="text-center"><i class="fa-solid fa-hourglass-half mr-1"></i>{{ getTimeForColor(opponentPlayer?.color) }}</p>
            </div>
            <div :class="getClassForPlayer(myPlayer.color)" class="bottom-12">
                <p>{{ myPlayer?.name }}</p>
                <div class="block w-10 aspect-square my-2 mx-auto" v-html="regenerate(myPlayer?.randomString)"></div>
                <p class="text-center"><i class="fa-solid fa-hourglass-half mr-1"></i>{{ getTimeForColor(myPlayer?.color) }}</p>


                <div class="emoji-container absolute bottom-0 left-[115%]">
                    <button
                        @click="showEmojisOption = !showEmojisOption"
                        class="w-12 h-12 flex items-center justify-center
                            bg-white/10 hover:bg-white/20
                            text-white text-xl
                            rounded-full shadow-lg
                            transition-all duration-200 active:scale-90"
                    >
                        ❤️
                    </button>

                    <!-- Emoji Popup -->
                    <transition name="fade">
                        <div
                            v-if="showEmojisOption"
                            class="absolute bottom-14 left-0
                                bg-white/90 backdrop-blur-md
                                p-3 rounded-2xl shadow-2xl
                                grid grid-cols-5 gap-2
                                w-40"
                        >
                            <span
                                v-for="(emoji, index) in emojiList"
                                :key="index"
                                @click="sendEmoji(emoji)"
                                class="cursor-pointer text-xl
                                    hover:scale-125 transition-transform duration-150"
                            >
                                {{ emoji }}
                            </span>
                        </div>
                    </transition>
                </div>

                <div class="gif-container absolute top-[0%] left-[115%]">
                    <button
                        @click="showGifsOption = !showGifsOption"
                        class="w-12 h-12 flex items-center justify-center
                            bg-white/10 hover:bg-white/20
                            text-white text-xl
                            rounded-full shadow-lg
                            transition-all duration-200 active:scale-90"
                    >
                        🤡
                    </button>

                    <!-- Gifs Popup -->
                     <transition name="fade">
                        
                        <div
                            v-if="showGifsOption"
                            class="absolute bottom-14 left-[-25vw] w-[80vw]
                                bg-white/90 backdrop-blur-md
                                p-3 rounded-2xl shadow-2xl"
                        >
                            <div class="grid grid-cols-3 gap-4 mt-6">
                                <img
                                    v-for="gif in gifs"
                                    :key="gif.id"
                                    :src="gif.images.fixed_height.url"
                                    class="w-full"
                                    @click="sendGif(gif)"
                                />
                            </div>
                            <h1 class="text-xl font-bold mb-4 text-black">Giphy Search</h1>

                            <input
                                v-model="query"
                                placeholder="Search GIFs..."
                                class="border p-2 mr-2 mb-2 text-black"
                            />

                            <button :disabled="gifSearchCount >= gifSearchLimit" @click="searchGifs" class="bg-blue-500 px-4 py-2">
                                Search({{ gifSearchLimit - gifSearchCount }})
                            </button>

                        </div>
                    </transition>
                </div>
            </div>
            <div class="p-3 bg-white/60 backdrop-blur-md border border-gray-300 rounded-xl shadow-md absolute bottom-12 right-4">
                <div class="text-center">
                    <i class="fa-solid fa-chess mr-1 "></i>
                     <span class="uppercase text-center font-large">
                        <template v-if="!replayModeOn">
                            {{ moveLog.length }}
                        </template>
                    </span>
                </div>
                <p class="text-center">{{ convertTime(totalTimeInSeconds) }}</p>
                <hr>
                <span class="text-center">#{{ roomCode }}</span>
            </div>

        </div>

    </div>
    <transition name="fade">
        <div
            v-if="showEmoji"
            class="fixed inset-0 flex items-center justify-center pointer-events-none"
            >
            <div class="text-7xl animate-scale">
                {{ recievedEmoji.emoji }}
            </div>
        </div>
    </transition>
    <transition>

        <div
            v-if="showGif"
            class="fixed inset-0 flex items-center justify-center pointer-events-none"
            >
            <div class="text-7xl animate-scale">
                <img
                    :src="recievedGif.gif.images.fixed_height.url"
                    class="w-full"
                />
                <!-- {{ recievedGif.gif }} -->
            </div>
        </div>
    </transition>

</template>

<script>
import db from './firebase.js';
import { randomNames } from './name.js';
// import gifSearch from "vue-gif-search";

export default {
    // components: {
    //     gifSearch
    // },
    name: 'ChessBoard',
    data() {
        return {
            // Board squares
            boardSquares: this.initBoardSquares(),
            // All pieces
            pieces: this.initPieces(),
            // Selected piece
            selected: null,
            currentTurn: 'white',
            winner: null,
            possibleMoves: [],

            promotionModal: false,
            promotionPiece: null,
            promotionMoveRecord: null,

            audioEnabled: true,
            moveAudio1: null,
            moveAudio2: null,
            toggleSound: false,

            moveLog: [],
            
            replayModeOn: false,
            replayModal: false,
            replayIndex: 0,
            replayLogInput: '',
            isAutoReplay: false,

            tempUCI: '',

            whiteTimeInSeconds: 0,
            blackTimeInSeconds: 0,
            timerInterval: null,

            draggingPiece: null,
            dragOffset: { x: 0, y: 0 },
            boardRect: null,

            // ----------------------------
            devMode: false,
            currentPage: 'before',

            defaultNumber: 2,
            maxPlayerNumber: 2,
            randomNames,

            firebaseRoomName: 'chess-rooms',
            roomOption: null,
            roomCode: null,
            tempRoomcode: null,
            generalData: null,

            username: null,
            onlineStatus: '',

            gameResults: [],
            previousGameResults: [],

            pickedRandomString: null,
            avatars: [],
            randomString: 0,

            isCheckingNow: false,
            gameMessage: "",
            isHost: false,
            developingMode: false,

            players: [],
            showEmojisOption: false,
            // emojiList: ['😊','😂','😍','👍','👎','🎉','😢','😡','🤔','🙌'],
            emojiList: [
                '😏','😈','🤡','💀','🧠','🧂','🐢','🐐','🔥','👀',
                '🙄','😴','😬','🫠','🎣','🤦','😮‍💨','🥱','🪦','🚩'
            ],
            recievedEmoji: [],
            showEmoji: false,

            showGifsOption: false,
            query: "",
            gifs: [],
            apiKey: "EirsrNXP0j8NitRZA94PkcLho9ylNlQ2",
            gifSearchLimit: 15,
            gifSearchCount: 0,
            recievedGif: [],
            showGif: false,
        }
    },
    mounted() {
        console.clear()

        // Create the audio objects
        this.moveAudio1 = new Audio('/move1.mp3')
        this.moveAudio2 = new Audio('/move2.mp3')

        this.generateAvatars();
        // this.currentPlayerIndex = 0

        this.username = this.getRandomName();
        
        this.devMode = true

        if (window.location.origin === 'https://nisino25-chess.netlify.app') {
            this.devMode = false
        }

        // this.startTimer()

    },
    methods: {
        // Create 8x8 squares
        initBoardSquares() {
            const squares = []
            for (let row = 0; row < 8; row++) {
                for (let col = 0; col < 8; col++) {
                    squares.push({ row, col })
                }
            }
            return squares
        },
        // Create initial pieces as objects
        initPieces() {
            const pieces = []

            // Black pieces
            const blackBack = ['rook','knight','bishop','queen','king','bishop','knight','rook']
            const blackPawns = Array(8).fill('pawn')

            blackBack.forEach((p, i) => pieces.push({ id: `b${i}`, type: p, row: 0, col: i, color: 'black',hasMoved: false  }))
            blackPawns.forEach((p, i) => pieces.push({ id: `bp${i}`, type: p, row: 1, col: i, color: 'black', hasMoved: false  }))

            // White pieces
            const whitePawns = Array(8).fill('pawn')
            const whiteBack = ['rook','knight','bishop','queen','king','bishop','knight','rook']
            // const whiteBack = ['rook','','','','king','','','rook']

            whitePawns.forEach((p, i) => pieces.push({ id: `wp${i}`, type: p, row: 6, col: i, color: 'white', hasMoved: false  }))
            whiteBack.forEach((p, i) => pieces.push({ id: `w${i}`, type: p, row: 7, col: i, color: 'white', hasMoved: false  }))

            return pieces
        },
        // Square color
        squareClass(row, col) {
            return (row + col) % 2 === 0
                ? 'bg-gray-400 border border-gray-400'  // light stone
                : 'bg-gray-600 border border-gray-600'  // dark stone
        },

        // Click piece
        selectPiece(piece) {
            if(this.winner) return;
            if (!this.selected) {
                if(this.currentTurn !== piece.color) {
                    alert("It's not your turn!")
                    return
                }
                if(this.myPlayer.color !== piece.color) {
                    alert("You can't move opponent's pieces!")
                    return
                }
                this.selected = piece
                this.possibleMoves = this.getPossibleMoves(piece)
            } else {
                // Move selected piece to clicked piece's square if different
                if (this.selected.id !== piece.id) {
                    this.movePiece(this.selected, piece.row, piece.col)
                }
                this.selected = null
                this.possibleMoves = []
            }
        },
        moveToTile(row, col) {
            if (this.selected) {
                this.movePiece(this.selected, row, col)
                this.selected = null
            }
            this.possibleMoves = []

        },
        // Move piece
        movePiece(piece, row, col) {
            const fromRow = piece.row;
            const fromCol = piece.col;

            // =========================
            // CASTLING FIRST
            // =========================
            if (
                piece.type === 'king' &&
                Math.abs(col - fromCol) === 2 &&
                row === fromRow
            ) {
                return this.tryCastling(piece, row, col);
            }
            // ❌ Not a legal move → stop
            if (!this.possibleMoves.some(m => m.row === row && m.col === col)) return false;


            const targetIdx = this.pieces.findIndex(p => p.row === row && p.col === col);
            const targetPiece = this.pieces[targetIdx];

            // Same color? stop
            if (targetIdx !== -1 && piece.color === targetPiece.color) return false;

            // Remove captured piece
            if (targetIdx !== -1) this.pieces.splice(targetIdx, 1);

            // Move piece
            piece.row = row;
            piece.col = col;
            piece.hasMoved = true;

            // Check pawn promotion
            let promotion = null;
            if (piece.type === 'pawn' && (row === 0 || row === 7)) {
                // Store the starting coords for UCI later
                piece.from = { row: piece.row, col: piece.col }  
                piece.row = row
                piece.col = col
                this.promotionPiece = piece
                this.promotionModal = true
                this.tempUCI = this.toUCI({
                    from: { row: fromRow, col: fromCol },
                    to: { row, col },
                });
                return
            }


            // Record move in UCI format
            const uci = this.toUCI({
                from: { row: fromRow, col: fromCol },
                to: { row, col },
                promotion
            });

            this.moveLog.push(uci);

            this.playMoveSound();

            // Check king capture
            if (targetPiece?.type === 'king') {
                this.winner = piece.color;
                alert(`${this.winner} wins!`);
                alert(this.moveLog.join(',')); // single line UCI log

                this.updateMoves();
                return;
            }


            this.updateMoves();


            // Switch turn
            this.currentTurn = this.currentTurn === 'white' ? 'black' : 'white';
            this.possibleMoves = [];
        },
        toUCI({ from, to, promotion }) {
            const cols = ['a','b','c','d','e','f','g','h'];
            const rows = ['8','7','6','5','4','3','2','1'];

            let uci = cols[from.col] + rows[from.row] + cols[to.col] + rows[to.row];
            if (promotion) uci += promotion.toLowerCase();
            return uci;
        },

        
        tryCastling(king, row, col) {

            const isKingSide = col > king.col;
            const rookCol = isKingSide ? 7 : 0;

            const rook = this.pieces.find(p =>
                p.row === row &&
                p.col === rookCol &&
                p.type === 'rook' &&
                p.color === king.color
            );

            if (!rook || king.hasMoved || rook.hasMoved) return false;

            // Check squares between king and rook are empty
            const betweenCols = isKingSide ? [5, 6] : [1, 2, 3];
            const isClear = betweenCols.every(c => !this.pieces.some(p => p.row === row && p.col === c));
            if (!isClear) return false;

            // TODO: Check king not in check and doesn't pass through check

            // ---- Perform castling ----
            king.col = col;
            rook.col = isKingSide ? col - 1 : col + 1;
            king.hasMoved = true;
            rook.hasMoved = true;

            // ---- LOG ----
            // Traditional chess notation for castling
            const castlingNotation = isKingSide ? 'O-O' : 'O-O-O';
            this.moveLog.push(castlingNotation);
            this.updateMoves();

            this.playMoveSound();

            this.currentTurn = this.currentTurn === 'white' ? 'black' : 'white';
            this.possibleMoves = [];

            return true;
        },

        getPieceAt(row, col) {
            return this.pieces.find(p => p.row === row && p.col === col)
        },

        rowColToSquare(row, col) {
            const files = ['a','b','c','d','e','f','g','h']
            const ranks = ['8','7','6','5','4','3','2','1']
            return files[col] + ranks[row]
        },
        
        isPossibleMove(row, col) {
            return this.possibleMoves.some(
            m => m.row === row && m.col === col
            )
        },
        getPossibleMoves(piece) {
            const moves = []
        
            // Helper: check if square is empty
            const isEmpty = (r, c) => !this.getPieceAt(r, c)
        
            // -----------------------
            // Bishop
            // -----------------------
            if (piece.type === 'bishop') {
                const directions = [
                    { r: 1, c: 1 }, { r: 1, c: -1 },
                    { r: -1, c: 1 }, { r: -1, c: -1 }
                ]
                directions.forEach(dir => {
                    let r = piece.row + dir.r
                    let c = piece.col + dir.c
                    while (r >= 0 && r < 8 && c >= 0 && c < 8) {
                        const target = this.getPieceAt(r, c)
                        if (!target) {
                            moves.push({ row: r, col: c })
                        } else {
                            if (target.color !== piece.color) moves.push({ row: r, col: c })
                            break
                        }
                        r += dir.r
                        c += dir.c
                    }
                })
            }
        
            // -----------------------
            // Rook
            // -----------------------
            else if (piece.type === 'rook') {
                const directions = [
                    { r: 1, c: 0 }, { r: -1, c: 0 },
                    { r: 0, c: 1 }, { r: 0, c: -1 }
                ]
                directions.forEach(dir => {
                    let r = piece.row + dir.r
                    let c = piece.col + dir.c
                    while (r >= 0 && r < 8 && c >= 0 && c < 8) {
                        const target = this.getPieceAt(r, c)
                        if (!target) {
                            moves.push({ row: r, col: c })
                        } else {
                            if (target.color !== piece.color) moves.push({ row: r, col: c })
                            break
                        }
                        r += dir.r
                        c += dir.c
                    }
                })
            }
        
            // -----------------------
            // Knight
            // -----------------------
            else if (piece.type === 'knight') {
                const directions = [
                    { r: 2, c: 1 }, { r: 2, c: -1 },
                    { r: -2, c: 1 }, { r: -2, c: -1 },
                    { r: 1, c: 2 }, { r: 1, c: -2 },
                    { r: -1, c: 2 }, { r: -1, c: -2 }
                ]
                directions.forEach(dir => {
                    const r = piece.row + dir.r
                    const c = piece.col + dir.c
                    if (r < 0 || r >= 8 || c < 0 || c >= 8) return
                    const target = this.getPieceAt(r, c)
                    if (target && target.color === piece.color) return
                    moves.push({ row: r, col: c })
                })
            }
        
            // -----------------------
            // Queen
            // -----------------------
            else if (piece.type === 'queen') {
                const directions = [
                    { r: 1, c: 0 }, { r: -1, c: 0 },
                    { r: 0, c: 1 }, { r: 0, c: -1 },
                    { r: 1, c: 1 }, { r: 1, c: -1 },
                    { r: -1, c: 1 }, { r: -1, c: -1 }
                ]
                directions.forEach(dir => {
                    let r = piece.row + dir.r
                    let c = piece.col + dir.c
                    while (r >= 0 && r < 8 && c >= 0 && c < 8) {
                        const target = this.getPieceAt(r, c)
                        if (!target) {
                            moves.push({ row: r, col: c })
                        } else {
                            if (target.color !== piece.color) moves.push({ row: r, col: c })
                            break
                        }
                        r += dir.r
                        c += dir.c
                    }
                })
            }
        
            // -----------------------
            // King (normal + castling)
            // -----------------------
            else if (piece.type === 'king') {
                const directions = [
                    { r: 1, c: 0 }, { r: -1, c: 0 },
                    { r: 0, c: 1 }, { r: 0, c: -1 },
                    { r: 1, c: 1 }, { r: 1, c: -1 },
                    { r: -1, c: 1 }, { r: -1, c: -1 }
                ]
                directions.forEach(dir => {
                    const r = piece.row + dir.r
                    const c = piece.col + dir.c
                    if (r < 0 || r >= 8 || c < 0 || c >= 8) return
                    const target = this.getPieceAt(r, c)
                    if (target && target.color === piece.color) return
                    moves.push({ row: r, col: c })
                })
        
                // Castling logic
                if (!piece.hasMoved) {
                    const row = piece.row
                    // King-side castling
                    const rookRight = this.getPieceAt(row, 7)
                    if (rookRight && rookRight.type === 'rook' && !rookRight.hasMoved) {
                        if (isEmpty(row, 5) && isEmpty(row, 6)) {
                            moves.push({ row, col: 6, castling: 'king-side' })
                        }
                    }
                    // Queen-side castling
                    const rookLeft = this.getPieceAt(row, 0)
                    if (rookLeft && rookLeft.type === 'rook' && !rookLeft.hasMoved) {
                        if (isEmpty(row, 1) && isEmpty(row, 2) && isEmpty(row, 3)) {
                            moves.push({ row, col: 2, castling: 'queen-side' })
                        }
                    }
                }
            }
        
            // -----------------------
            // Pawn
            // -----------------------
            else if (piece.type === 'pawn') {
                const dir = piece.color === 'white' ? -1 : 1
                const startRow = piece.color === 'white' ? 6 : 1
                const oneStepRow = piece.row + dir
                if (isEmpty(oneStepRow, piece.col)) {
                    moves.push({ row: oneStepRow, col: piece.col })
                    const twoStepRow = piece.row + dir * 2
                    if (piece.row === startRow && isEmpty(twoStepRow, piece.col)) {
                        moves.push({ row: twoStepRow, col: piece.col })
                    }
                }
                [-1, 1].forEach(dc => {
                    const r = piece.row + dir
                    const c = piece.col + dc
                    if (r < 0 || r >= 8 || c < 0 || c >= 8) return
                    const target = this.getPieceAt(r, c)
                    if (target && target.color !== piece.color) {
                        moves.push({ row: r, col: c })
                    }
                })
            }
        
            return moves
        },
        isLastMove(row,col){
            if(this.moveLog.length === 0) return false;

            const lastUCI = this.moveLog[this.moveLog.length -1];

            // CASTLING
            if(lastUCI === 'O-O' || lastUCI === 'O-O-O'){
                const isKingSide = lastUCI === 'O-O';
                const kingRow = this.currentTurn === 'white' ? 0 : 7; // since turn already switched
                const kingCol = isKingSide ? 6 : 2;
                const rookCol = isKingSide ? 5 : 3;

                return (row === kingRow && col === kingCol) || (row === kingRow && col === rookCol);
            }

            const cols = ['a','b','c','d','e','f','g','h'];
            const rows = ['8','7','6','5','4','3','2','1'];

            // const toCol = cols.indexOf(lastUCI[2]);
            // const toRow = rows.indexOf(lastUCI[3]);

            return (
                (row === rows.indexOf(lastUCI[3]) && col === cols.indexOf(lastUCI[2]))) 
                || 
                (row === rows.indexOf(lastUCI[1]) && col === cols.indexOf(lastUCI[0]));
        },

        undoMove() {
            // Remove the last move from log
            console.log(this.moveLog)
            if (!this.moveLog.length) return;
            if(this.replayModeOn) this.replayIndex--;
            this.moveLog.pop();



            

            // Replay all moves in log
            this.replayAllMoves(this.moveLog)
            

            // Switch turn back
            this.currentTurn = this.currentTurn === 'white' ? 'black' : 'white';
            this.winner = null;
            this.possibleMoves = [];

            this.updateMoves()
        },

        replayAllMoves(referMoves){
            this.moveLog = referMoves;

            // Reset board completely
            this.pieces = this.initPieces();

            // Temporary turn tracker for replaying moves
            let tempTurn = 'white';

            const promotionMap = {
                r: 'rook',
                k: 'knight',
                b: 'bishop',
                q: 'queen',
            };
            referMoves.forEach(uci => {
                if (uci === 'O-O' || uci === 'O-O-O') {
                    // CASTLING DETECTED
                    const isKingSide = uci === 'O-O';
                    const row = tempTurn === 'white' ? 7 : 0; // king's row
                    const king = this.pieces.find(p => p.type === 'king' && p.color === tempTurn);
                    const rook = this.pieces.find(p => p.type === 'rook' && p.color === tempTurn && (isKingSide ? p.col === 7 : p.col === 0));

                    // move king
                    king.col = isKingSide ? 6 : 2;
                    king.row = row;
                    king.hasMoved = true;

                    // move rook
                    rook.col = isKingSide ? 5 : 3;
                    rook.row = row;
                    rook.hasMoved = true;

                    tempTurn = tempTurn === 'white' ? 'black' : 'white';
                    return; // skip the rest of this iteration
                }

                const cols = ['a','b','c','d','e','f','g','h'];
                const rows = ['8','7','6','5','4','3','2','1'];

                const fromCol = cols.indexOf(uci[0]);
                const fromRow = rows.indexOf(uci[1]);
                const toCol = cols.indexOf(uci[2]);
                const toRow = rows.indexOf(uci[3]);
                const promotion = uci[4] || null;

                const piece = this.pieces.find(p => p.row === fromRow && p.col === fromCol);
                if (!piece) return;

                piece.row = toRow;
                piece.col = toCol;

                if (promotion) {
                    piece.type = promotionMap[promotion];
                }

                // Remove captured piece automatically if destination is occupied by enemy
                const targetIdx = this.pieces.findIndex(p => p.row === toRow && p.col === toCol && p.id !== piece.id);
                if (targetIdx !== -1) this.pieces.splice(targetIdx, 1);

                piece.hasMoved = true;
                // Switch temp turn
                tempTurn = tempTurn === 'white' ? 'black' : 'white';
            });
            this.currentTurn = tempTurn;

            this.playMoveSound();
        },

        resetBoard(forReplay) {
            if (!confirm('Reset the game? This will clear all moves.')) return

            this.pieces = this.initPieces()
            this.selected = null
            this.currentTurn = 'white'
            this.winner = null
            this.possibleMoves = []
            this.moveLog = []

            if(!forReplay){
                this.replayIndex = 0;
                this.replayModeOn = false;
                this.replayMoves = [];
            }

            this.whiteTimeInSeconds = 0;
            this.blackTimeInSeconds = 0;

            return;
        },
        promotePawn(type) {
            if (!this.promotionPiece) return

            // Update piece type
            this.promotionPiece.type = type

            const uci = this.tempUCI + type[0].toLowerCase()
            this.moveLog.push(uci)
            this.updateMoves();

            this.tempUCI = ''

            // Close modal
            this.promotionPiece = null
            this.promotionModal = false

            // Switch turn
            this.currentTurn = this.currentTurn === 'white' ? 'black' : 'white'
        },

        enableAudio() {
            this.audioEnabled = true

            // initial Audio objects just to unlock
            this.moveAudio1 = new Audio('/move1.mp3')
            this.moveAudio2 = new Audio('/move2.mp3')

            // play once to unlock
            this.moveAudio1.play().catch(()=>{})
        },
        playMoveSound() {

            if (!this.audioEnabled) return

            // Alternate between the two audio sources
            const src = this.toggleSound ? '/move1.mp3' : '/move2.mp3'

            const sound = new Audio(src)
            sound.volume = 0.5
            sound.play().catch(() => {})

            this.toggleSound = !this.toggleSound
        },

        startReplay() {
            if (!this.replayLogInput) return

            const moves = this.replayLogInput
                .split(',')
                .map(m => m.trim())
                .filter(Boolean)

            this.replayModeOn = true
            this.replayModal = false
            this.replayMoves = moves

            this.resetBoard(true)
            this.replayIndex = 0

            this.playReplay()
        },
        applyUCIMove(uci) {
            // CASTLING
            if (uci === 'O-O' || uci === 'O-O-O') {
                const isKingSide = uci === 'O-O';
                const row = this.currentTurn === 'white' ? 7 : 0;
                const king = this.pieces.find(p => p.type === 'king' && p.color === this.currentTurn);
                const rook = this.pieces.find(
                    p => p.type === 'rook' && p.color === this.currentTurn && (isKingSide ? p.col === 7 : p.col === 0)
                );

                if (!king || !rook) return;

                king.col = isKingSide ? 6 : 2;
                king.row = row;
                king.hasMoved = true;

                rook.col = isKingSide ? 5 : 3;
                rook.row = row;
                rook.hasMoved = true;

                this.moveLog.push(uci);
                this.currentTurn = this.currentTurn === 'white' ? 'black' : 'white';
                return;
            }

            const cols = ['a','b','c','d','e','f','g','h']
            const rows = ['8','7','6','5','4','3','2','1']

            const fromCol = cols.indexOf(uci[0])
            const fromRow = rows.indexOf(uci[1])
            const toCol = cols.indexOf(uci[2])
            const toRow = rows.indexOf(uci[3])
            const promotion = uci.length === 5 ? uci[4] : null

            const piece = this.pieces.find(
                p => p.row === fromRow && p.col === fromCol
            )
            if (!piece) return

            // capture
            const targetIdx = this.pieces.findIndex(
                p => p.row === toRow && p.col === toCol
            )
            if (targetIdx !== -1) {
                this.pieces.splice(targetIdx, 1)
            }

            piece.row = toRow
            piece.col = toCol
            piece.hasMoved = true

            // promotion
            if (promotion) {
                const map = {
                    q: 'queen',
                    r: 'rook',
                    b: 'bishop',
                    n: 'knight'
                }
                piece.type = map[promotion]
            }

            this.moveLog.push(uci)
            this.currentTurn = this.currentTurn === 'white' ? 'black' : 'white'
        },
        playReplay() {
            // stop immediately if paused
            if (!this.isAutoReplay) return

            // stop if finished
            if (this.replayIndex >= this.replayMoves.length) {
                this.isAutoReplay = false
                return
            }

            this.applyUCIMove(this.replayMoves[this.replayIndex])
            this.replayIndex++

            setTimeout(() => {
                this.playReplay()
            }, 500)
        },
        nextMove() {
            if (this.replayIndex >= this.replayMoves.length) return
            this.applyUCIMove(this.replayMoves[this.replayIndex])
            this.replayIndex++
        },
        toggleAutoReplay() {
            if (this.isAutoReplay) {
                // pause
                this.isAutoReplay = false
            } else {
                // play
                this.isAutoReplay = true
                this.playReplay()
            }
        },

        startTimer() {
            if (this.timerInterval) return // safety

            this.timerInterval = setInterval(() => {
                if (this.currentTurn === 'white') {
                    this.whiteTimeInSeconds++
                } else {
                    this.blackTimeInSeconds++
                }
            }, 1000)
        },
        stopTimer() {
            clearInterval(this.timerInterval)
            this.timerInterval = null
        },
        convertTime(seconds) {
            const minutes = Math.floor(seconds / 60)
            const remainingSeconds = seconds % 60

            return `${minutes}:${remainingSeconds.toString().padStart(2, '0')}`
        },
        getTimeForColor(color) {
            if(color === 'white') {
                return this.convertTime(this.whiteTimeInSeconds)
            } else {
                return this.convertTime(this.blackTimeInSeconds)
            }
        },

        getClassForPlayer(color) {
            return [
                'block p-2 border rounded-xl shadow-md absolute text-center left-4 w-[6em]',
                color === 'white'
                    ? 'bg-white/80 backdrop-blur-md text-black'
                    : 'bg-black/80 backdrop-blur-md text-white',

                this.currentTurn === color
                    ? 'ring-4 ring-yellow-400 shadow-lg shadow-yellow-300/50'
                    : ''
            ].join(' ')
        },

        // ----------------------------
        randomName(){
            this.username = this.getRandomName();
        },
        retriveCode(){
            this.tempRoomcode = localStorage.getItem('latestRoomCode') || 'No room code found'
        },
        nextRoomOption(option){
            if(!this.devMode){
                if(option == 'create'){
                    // this.createARoom()
                    this.roomOption = 'create'
                }
                if(option == 'join'){
                    this.retriveCode();
                    this.roomOption = 'join'
                    // this.joinARoom()
                }
                return
            }
            if(option == 'create'){
                this.createARoom()
            }
            if(option == 'join'){
                this.retriveCode();
                this.joinARoom()
            }
        },
        async createARoom() {
            if (this.roomCode) return;


            if(!this.developingMode){
                const ok = window.confirm('Start a new room?')
                if (!ok) return
            }
            // if(!this.username) return;

            let isUnique = false;

            // Generate a unique room code
            while (!isUnique) {
                this.tempRoomcode = Math.floor(10000 + Math.random() * 90000);
                const docRef = db.collection(this.firebaseRoomName).doc(`${this.tempRoomcode}`);
                const doc = await docRef.get();
                if (!doc.exists) {
                    isUnique = true;
                }
            }

            // console.log(this.roomCode)
            this.roomCode = this.tempRoomcode
            localStorage.setItem('latestRoomCode', this.roomCode)

            this.players = [
                {
                name:this.username,
                randomString: this.randomString,
                }
            ]

            const ref = db.collection(this.firebaseRoomName)
            ref.doc(`${this.roomCode}`).set({
                games: JSON.stringify([{ gameStatus: 'waiting' }]),
                players: this.players,
                onlineStatus: 'waiting',
                moveLog: [],
            })

            this.roomOption = 'create'
            this.onlineStatus = 'waiting'
            this.isHost = true
            await this.reciveTheData()
        },

        async joinARoom() {

            this.roomOption = 'join'
            const docRef = db.collection(this.firebaseRoomName).doc(`${this.tempRoomcode}`);

            try {
            const doc = await docRef.get();
            if (doc.exists) {
                if(doc.data().onlineStatus == 'playing') return alert('This room is closed.')

                this.players = doc.data().players;
                this.onlineStatus = doc.data().players;

                if (!this.players.includes(this.username)) {
                this.players.push(
                    {
                        name:this.username,
                        randomString: this.randomString,
                    }
                );
                this.roomCode = this.tempRoomcode

                }

                await docRef.update({
                    players: this.players,
                });
                this.reciveTheData();
            } else {
                console.log('No such document!');
            }
            } catch (error) {
            console.log('Error getting document:', error);
            }
        },

        reciveTheData(){
            
            db.collection(this.firebaseRoomName).doc(`${this.roomCode}`)
            .onSnapshot((doc) => {
            
            this.generalData = doc.data()
            
            // joining room and wait until it closes
            // if(this.currentPage == 'before'){
            this.players = this.generalData?.players
            let oldStatus = this.onlineStatus
            this.onlineStatus = this.generalData?.onlineStatus

            this.winner = this.generalData?.winner


            if(this.recievedEmoji.length !== (this.generalData?.emojiList?.length || 0)) {
                this.displayEmoji(this.generalData.emojiList)
            }

            if(this.recievedGif.length !== (this.generalData?.gifList?.length || 0)) {
                this.displayGif(this.generalData.gifList)
            }
            // }

            if(oldStatus !== 'playing' && this.onlineStatus == 'playing' && !this.isHost){
                this.startTimer() 
            }


            if(this.onlineStatus == 'playing' || this.onlineStatus == 'distributing') {
                this.currentPage = 'game'
                if(this.moveLog.length !== this.generalData?.moveLog?.length) this.replayAllMoves(this.generalData?.moveLog || [])
            }
            
            })
        },
        synchMoveLog(newMoveLog) {
            newMoveLog.forEach(move => {
                this.applyUCIMove(move)
            })

            this.moveLog = newMoveLog
        },

        async closeTheRoom(){

            if(this.players.length !== 2) return

            const colors = ['black', 'white']

            // shuffle
            colors.sort(() => Math.random() - 0.5)

            this.players.forEach((player, index) => {
                player.color = colors[index]
            })

            this.currentPage = 'game';

            // this.gameResults = []

            this.onlineStatus = 'playing'

            this.startTimer() 

            const ref = db.collection(this.firebaseRoomName)
            ref.doc(`${this.roomCode}`).update({
                winner: '',
                players: this.players,
                onlineStatus: this.onlineStatus,
                emojiList: [],
                gifList: [],
            })
        },

        generateAvatar() {
            const randomString = Math.random().toString();
            return {
                avatar: window.multiavatar(randomString),
                randomString: randomString
            };
        },
        generateAvatarSeed() {
            return Math.random().toString(36).slice(2);
        },

        regenerate(randomString) {
            return window.multiavatar(randomString);
        },

        generateAvatars() {
            this.tempAvatarCode = null;

            this.avatars = [1, 2, 3, 4, 5,6,7,8,9].map(() => {
            const randomString = Math.random().toString();
            return {
                avatar: window.multiavatar(randomString),
                randomString: randomString
            };
            });
        },
        getRandomName() {
            let randomName;
            // do {
                const randomIndex = Math.floor(Math.random() * this.randomNames.length);
                randomName = this.randomNames[randomIndex];
            // } while (this.players.some(player => player.name === randomName));

            // this.generateAvatars();

            return randomName;
        },

        updateMoves(){
            console.log('Updating winner in Firestore...', this.winner)
            const ref = db.collection(this.firebaseRoomName)
            ref.doc(`${this.roomCode}`).update({
                moveLog: this.moveLog,
                winner: this.winner,
            })
        },
        sendEmoji(emoji) {

            const newEmojiList = [...(this.generalData?.emojiList || []), { emoji, sender: this.username }]

            const ref = db.collection(this.firebaseRoomName)
            ref.doc(`${this.roomCode}`).update({
                emojiList: newEmojiList,
            })

            this.showEmojisOption = false;
        },
        displayEmoji(emojiList) {
            const latest = emojiList[emojiList.length - 1]

            this.recievedEmoji = latest
            this.showEmoji = true

            setTimeout(() => {
                this.showEmoji = false
            }, 2500)
        },

        async searchGifs() {
            if (!this.query) return;
            // if(this.gifSearchCount >= this.gifSearchLimit) return;

            const url = `https://api.giphy.com/v1/gifs/search?api_key=${this.apiKey}&q=${this.query}&limit=9`;
            this.gifSearchCount++;

            try {
                const res = await fetch(url);
                const data = await res.json();
                this.gifs = data.data;
            } catch (err) {
                console.error(err);
            }
        },
        sendGif(gif) {

            const newGifList = [...(this.generalData?.gifList || []), { gif, sender: this.username }]

            const ref = db.collection(this.firebaseRoomName)
            ref.doc(`${this.roomCode}`).update({
                gifList: newGifList,
            })

            this.showGifsOption = false;
        },
        displayGif(gifList) {
            const latest = gifList[gifList.length - 1]

            this.recievedGif = latest
            console.log(this.recievedGif)
            this.showGif = true

            setTimeout(() => {
                this.showGif = false
            }, 3500)
        },

        

    },

    computed: {
        totalTimeInSeconds() {
            return this.whiteTimeInSeconds + this.blackTimeInSeconds;
        },


        readyToPlay() {
          const namePattern = /^[^\s!@#$%^&*(),.?":{}|<>]+$/;

          // Check if the username is valid
          return namePattern.test(this.username) && this.username?.trim() !== '';
        },

        myPlayer(){
            return this.players.find(p => p.name === this.username)
        },

        opponentPlayer(){
            return this.players.find(p => p.name !== this.username)
        }
    },
    // watch: {
    //     moveLog: {
    //         handler(newVal) {
    //             console.log('moveLog changed', newVal)

    //             this.updateMoves()
    //         },
    //         deep: true
    //     }
    // }

}
</script>

<style>
    html,
    body {
        margin: 0;
        padding: 0;
        height: 100vh;
        overflow: hidden;

        background-color: #1e293b !important;
    }

    .piece {
        width: 12.5%;
        height: 12.5%;
        display: flex;
        align-items: center;
        justify-content: center;
    }

    @keyframes blink {
        0%, 100% {
            opacity: 0.4;
            background: gold;
        }
        50% {
            opacity: 0.9;
        }
    }

    .blink {
        animation: blink 1.2s ease-in-out infinite;
    }

    .square--last-move {
        background: rgba(50, 120, 220, 0.6) !important;

    }

    .fade-enter-active,
    .fade-leave-active {
        transition: opacity 0.6s ease;
    }

    .fade-enter-from,
    .fade-leave-to {
        opacity: 0;
    }

    .animate-scale {
        animation: pop 0.3s ease;
    }

    @keyframes pop {
        0% { transform: scale(0.6); }
        100% { transform: scale(1); }
    }


</style>
