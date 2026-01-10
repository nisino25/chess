<template>
    <div class="p-4 flex flex-col items-center">
        <h2 class="mb-4 text-lg font-medium">Current: {{ currentTurn }}'s turn</h2>
        <h3 v-if="winner" class="font-xl font-bold font-red">Winner: {{ winner }}</h3>

        <div class=" w-[92.5vw] aspect-square absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 border-4 border-gray-200 shadow-lg">
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
        class="piece absolute text-7xl transition-all duration-300 cursor-pointer pointer-events-auto"
        :style="{ top: `${piece.row * 12.5}%`, left: `${piece.col * 12.5}%` }"
        @click.stop="selectPiece(piece)"
        :class="selected && selected.id === piece.id ? 'ring-4 ring-indigo-400' : ''"
    >
        {{ piece.type }}
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
            possibleMoves: [] 
        }
    },
    mounted() {
        console.clear()
        console.log('Initial pieces:', this.pieces)

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
            const blackBack = ['♜','♞','♝','♛','♚','♝','♞','♜']
            const blackPawns = Array(8).fill('♟︎')
            blackBack.forEach((p, i) => pieces.push({ id: `b${i}`, type: p, row: 0, col: i, color: 'black' }))
            blackPawns.forEach((p, i) => pieces.push({ id: `bp${i}`, type: p, row: 1, col: i, color: 'black' }))

            // White pieces
            const whitePawns = Array(8).fill('♙')
            const whiteBack = ['♖','♘','♗','♕','♔','♗','♘','♖']
            whitePawns.forEach((p, i) => pieces.push({ id: `wp${i}`, type: p, row: 6, col: i, color: 'white' }))
            whiteBack.forEach((p, i) => pieces.push({ id: `w${i}`, type: p, row: 7, col: i, color: 'white' }))

            return pieces
        },
        // Square color
        squareClass(row, col) {
            return (row + col) % 2 === 0
                ? 'bg-amber-100 border border-gray-300'
                : 'bg-slate-400 border border-gray-300'
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
            console.log('moveToTile', row, col);
            if (this.selected) {
                    this.movePiece(this.selected, row, col)
                    this.selected = null
            }
        },
        // Move piece
        movePiece(piece, row, col) {
            // Check if another piece is on that square
            const targetIdx = this.pieces.findIndex(p => p.row === row && p.col === col)
            const targetPiece = this.pieces[targetIdx]

            if (targetIdx !== -1) {
                // Capture
                if(this.selected.color === targetPiece.color) return false
                this.pieces.splice(targetIdx, 1)
            }
            piece.row = row
            piece.col = col
            // ✅ Check if king is captured
            if (targetPiece?.type === '♔' || targetPiece?.type === '♚') {
                this.winner = piece.color
                alert(`${this.winner} wins!`)
                return
            }
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
    }
}
</script>

<style scoped>
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
