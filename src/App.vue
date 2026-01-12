<template>
    <div class="p-4 flex flex-col items-center">
        <div class="flex gap-4 items-center mb-4">
            <div class="p-4 flex flex-col items-center">
                 <div
                     class="flex items-center justify-center gap-2 px-4 py-2 rounded-full text-sm font-medium shadow-sm transition"
                     :class="currentTurn === 'white'
                         ? 'bg-gray-100 text-gray-900 border border-gray-300'
                         : 'bg-slate-900 text-white border border-slate-700'"
                 >
                     <i class="fa-solid fa-chess"></i>
                     <span class="uppercase">{{ currentTurn }}'s turn</span>
                 </div>
             </div>

             <div class="flex gap-4 p-4 bg-white/80 backdrop-blur-md border border-gray-300 rounded-xl shadow-md">
                 <button
                     @click="undoMove"
                     class="px-4 py-2 bg-indigo-500 text-white rounded hover:bg-indigo-600 transition"
                 >
                     Undo Move
                 </button>
                 <button
                     @click="resetBoard"
                     class="px-4 py-2 bg-red-500 text-white rounded  transition"
                 >
                     <i class="fa-solid fa-trash"></i>
                 </button>
             </div>
        </div> 




        <h3 v-if="winner" class="font-xl font-bold font-red">Winner: {{ winner }}</h3>
        <hr>

        <div class="w-[92.5vw] aspect-square absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 border-4 border-gray-400 shadow-lg">
            <!-- Board squares -->
            <div class="grid grid-cols-8 grid-rows-8 w-full h-full">
                <div
                    v-for="square in boardSquares"
                    :key="`${square.row}-${square.col}`"
                    :class="[
                        squareClass(square.row, square.col),
                        isPossibleMove(square.row, square.col) ? 'blink' : ''
                    ]"
                    @click="moveToTile(square.row, square.col)"
                ></div>
            </div>



            <!-- Pieces layer -->
            <div class="absolute top-0 left-0 w-full h-full pointer-events-none">
                <div
                    v-for="piece in pieces"
                    :key="piece.id"
                    class="piece absolute text-5xl transition-all duration-300 cursor-pointer pointer-events-auto"
                    :style="{ top: `${piece.row * 12.5}%`, left: `${piece.col * 12.5}%` }"
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
        <div
            v-if="promotionModal"
            class="fixed inset-0 bg-black/50 flex items-center justify-center z-50"
        >
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

            moveSound: null,


            moveLog: [] ,
        }
    },
    mounted() {
        console.clear()
        console.log('Initial pieces:', this.pieces);

        const context = new (window.AudioContext || window.webkitAudioContext)()
        fetch('/move.mp3')
            .then(res => res.arrayBuffer())
            .then(buffer => context.decodeAudioData(buffer))
            .then(decoded => { this.moveSoundBuffer = decoded; this.audioContext = context })

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

            blackBack.forEach((p, i) => pieces.push({ id: `b${i}`, type: p, row: 0, col: i, color: 'black' }))
            blackPawns.forEach((p, i) => pieces.push({ id: `bp${i}`, type: p, row: 1, col: i, color: 'black' }))

            // White pieces
            const whitePawns = Array(8).fill('pawn')
            const whiteBack = ['rook','knight','bishop','queen','king','bishop','knight','rook']

            whitePawns.forEach((p, i) => pieces.push({ id: `wp${i}`, type: p, row: 6, col: i, color: 'white' }))
            whiteBack.forEach((p, i) => pieces.push({ id: `w${i}`, type: p, row: 7, col: i, color: 'white' }))

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
            // Check if another piece is on that square
            const targetIdx = this.pieces.findIndex(p => p.row === row && p.col === col)
            const targetPiece = this.pieces[targetIdx]

            // Same color? stop
            if (targetIdx !== -1 && this.selected.color === targetPiece.color) return false

            // Remove captured piece immediately
            if (targetIdx !== -1) {
                this.pieces.splice(targetIdx, 1)
            }

            const moveRecord = {
                pieceId: piece.id,
                from: { row: piece.row, col: piece.col },
                to: { row, col },
                captured: targetPiece
                    ? { pieceId: targetPiece.id, type: targetPiece.type, color: targetPiece.color }
                    : null,
                turn: piece.color,
                promotionBefore: piece.type,
                promotion: null
            }

            piece.row = row
            piece.col = col

            // Check for pawn promotion **after capturing**
            if (piece.type === 'pawn' && (row === 0 || row === 7)) {
                this.promotionPiece = piece
                this.promotionMoveRecord = moveRecord
                this.promotionModal = true
                return
            }

            // Add move to log AFTER promotion check
            this.moveLog.push(moveRecord)

            // Check for king capture
            if (targetPiece?.type === 'king') {
                this.winner = piece.color
                alert(`${this.winner} wins!`)
                return
            }

            this.playMoveSound();


            this.currentTurn = this.currentTurn === 'white' ? 'black' : 'white'
            this.possibleMoves = []
        },

        isPossibleMove(row, col) {
            return this.possibleMoves.some(
            m => m.row === row && m.col === col
            )
        },
        getPossibleMoves(piece) {
            // TEMP: allow move anywhere (for testing)
            const moves = []

            for (let row = 0; row < 8; row++) {
                for (let col = 0; col < 8; col++) {
                    if (row !== piece.row || col !== piece.col) {
                        moves.push({ row, col })
                    }
                }
            }

            return moves
        },
        undoMove() {
            const lastMove = this.moveLog.pop()
            if (!lastMove) return

            // Move piece back
            const piece = this.pieces.find(p => p.id === lastMove.pieceId)
            piece.row = lastMove.from.row
            piece.col = lastMove.from.col

            // Revert promotion if any
            if (lastMove.promotion) {
                piece.type = lastMove.promotionBefore
            }

            // Restore captured piece if any
            if (lastMove.captured) {
                this.pieces.push({
                    id: lastMove.captured.pieceId,
                    type: lastMove.captured.type,
                    color: lastMove.captured.color,
                    row: lastMove.to.row,
                    col: lastMove.to.col
                })
            }

            this.currentTurn = lastMove.turn
            this.winner = null
        },

        resetBoard() {
            if (!confirm('Reset the game? This will clear all moves.')) return

            this.pieces = this.initPieces()
            this.selected = null
            this.currentTurn = 'white'
            this.winner = null
            this.possibleMoves = []
            this.moveLog = []
        },

        promotePawn(type) {
            if (!this.promotionPiece || !this.promotionMoveRecord) return

            // Save promotion info in move record
            this.promotionMoveRecord.promotion = type

            // Update piece
            this.promotionPiece.type = type

            // Push move to log AFTER promotion
            this.moveLog.push(this.promotionMoveRecord)

            // Close modal
            this.promotionPiece = null
            this.promotionMoveRecord = null
            this.promotionModal = false

            // Switch turn
            this.currentTurn = this.currentTurn === 'white' ? 'black' : 'white'
        },
        playMoveSound() {
            const source = this.audioContext.createBufferSource()
            source.buffer = this.moveSoundBuffer
            source.connect(this.audioContext.destination)
            source.start(0)
        },
    }
}
</script>

<style scoped>
html{
    background-color: #4fc08d;
}

body {
  margin:0;
  padding:0;
  height: 100vh;
  overflow: hidden;
  
  
  /* background-color: red; */
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
    }
    50% {
        opacity: 0.9;
    }
}

.blink {
    animation: blink 1.2s ease-in-out infinite;
}

</style>
