<template>
    <div class="p-4 flex flex-col items-center">
        <div class="flex gap-4 items-center mb-4">
            <div class="p-4 flex flex-col items-center">
                 <div
                     class="flex items-center justify-center gap-2 px-4 py-2 rounded-full text-medium shadow-sm transition"
                     :class="currentTurn === 'white'
                         ? 'bg-gray-100 text-gray-900 border border-gray-300'
                         : 'bg-slate-900 text-white border border-slate-700'"
                 >
                     <i class="fa-solid fa-chess"></i>
                     <span class="uppercase text-center font-large">
                        <template v-if="!replayModeOn">
                            {{ moveLog.length }}
                        </template>
                        <template v-else><br>{{ replayIndex }}/{{ replayMoves.length }}</template>
                        <!-- {{ replayMoves?.slice(replayIndex).join(', ') }} -->
                    </span>
                 </div>
             </div>

             <div class="grid grid-cols-3 gap-4 p-4 bg-white/80 backdrop-blur-md border border-gray-300 rounded-xl shadow-md">
                <button
                    @click="undoMove"
                    class="px-4 py-2 bg-indigo-500 text-white rounded hover:bg-indigo-600 transition"
                >
                <i class="fa-solid fa-rotate-left"></i>
                </button>
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
        </div> 




        <h3 v-if="winner" class="font-xl font-bold font-red">Winner: {{ winner }}</h3>
        <hr>

        <div class="max-w-[725px] w-[92.5vw] aspect-square absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 border-4 border-gray-800 shadow-lg">
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
                    :class="selected && selected.id === piece.id ? 'ring-4 ring-indigo-400' : ''"
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

        <div class="timer-container flex justify-between mt-8 absolute px-8 bottom-8 w-full">
            <div class="p-3 bg-white/80 backdrop-blur-md border border-gray-300 rounded-xl shadow-md">
                <span>White <i class="fa-solid fa-hourglass-half"></i></span>
                <p class="text-center">{{ convertTime(whiteTimeInSeconds) }}</p>
            </div>
            <div class="p-3 bg-gray/200 backdrop-blur-md border border-gray-300 rounded-xl shadow-md">
                <span>Total <i class="fa-solid fa-hourglass-half"></i></span>
                <p class="text-center">{{ convertTime(totalTimeInSeconds) }}</p>
            </div>
            <div class="p-3 bg-black/80 backdrop-blur-md border border-gray-300 rounded-xl shadow-md text-white">
                <span>Black <i class="fa-solid fa-hourglass-half"></i></span>
                <p class="text-center">{{ convertTime(blackTimeInSeconds) }}</p>
            </div>
        </div>

    </div>
</template>

<script>
export default {
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
            pgnMoves: [],  
            
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
        }
    },
    mounted() {
        console.clear()

        // Create the audio objects
        this.moveAudio1 = new Audio('/move1.mp3')
        this.moveAudio2 = new Audio('/move2.mp3')

        this.moveAudio1.volume = 0.5
        this.moveAudio2.volume = 0.5

        // Safari/iOS requires a user interaction first
        // So we wait for the first click to unlock the audio
        const unlockAudio = () => {
            // Play a tiny silent sound to unlock
            this.moveAudio1.play().catch(() => {})
            window.removeEventListener('click', unlockAudio)
        }
        window.addEventListener('click', unlockAudio)

        this.startTimer()

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
            if (!this.selected) {
                if(this.currentTurn !== piece.color) {
                    alert("It's not your turn!")
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
                return;
            }

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

            // Reset board completely
            this.pieces = this.initPieces();


            // Temporary turn tracker for replaying moves
            let tempTurn = 'white';

            const promotionMap = {
                q: 'queen',
                r: 'rook',
                b: 'bishop',
                n: 'knight'
            };

            // Replay all moves in log
            this.moveLog.forEach(uci => {
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

            // Switch turn back
            this.currentTurn = this.currentTurn === 'white' ? 'black' : 'white';
            this.winner = null;
            this.possibleMoves = [];
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

    },
    computed: {
        totalTimeInSeconds() {
            return this.whiteTimeInSeconds + this.blackTimeInSeconds;
        }
    },
}
</script>

<style>
    html,
    body {
        margin: 0;
        padding: 0;
        height: 100vh;
        overflow: hidden;
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

</style>
