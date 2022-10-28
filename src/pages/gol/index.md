---
title: Conway's Game of Life
version: 0.1.0
description: Conway's Game of Life in sCrypt
---

Rules:
1. Any live cell with fewer than two live neighbours dies, as if by needs caused by underpopulation.
2. Any live cell with more than three live neighbours dies, as if by overcrowding.
3. Any live cell with two or three live neighbours lives, unchanged, to the next generation.
4. Any dead cell with exactly three live neighbours cells will come to life.

The generation of the game is stored as the [state](https://by-example.scrypt.io/stateful-contract/) of the contract.

```javascript
import "arrayUtil.scrypt";

// Conway Game Of Life on a board of N * N
contract GameOfLife {
    static const int N = 5;
    // effctively we play on a grid of (N+2) * (N+2) without handling boundary cells
    static const int BOARD_SIZE = GameOfLife.N + 2;
    //static const int XY = BOARD_SIZE * BOARD_SIZE;  <- error: invalid array size when run #serializeState
    static const int XY = 49; 
    static int LIVE = 1;
    static int DEAD = 0;
    static const int LOOP_NEIGHBORS = 3;

    @state
    int[XY] board;

    public function play(int amount, SigHashPreimage txPreimage) {
        // make the move
        this.board = GameOfLife.evolve(this.board);

        // update state: next turn & next board
        require(this.propagateState(txPreimage, amount));
    }

    static function evolve(int[XY] oldBoard) : int[XY] {
        int[XY] newBoard = oldBoard;

        int i = 1;
        loop (GameOfLife.N) {
            int j = 1;
            loop (GameOfLife.N) {
                int nextState = GameOfLife.next(oldBoard, i, j);
                newBoard[GameOfLife.index(i, j)] = nextState;
                
                j++;
            }

            i++;
        }

        return newBoard;
    }

    static function next(int[XY] oldBoard, int row, int col) : int {
        // number of neighbors alive
        int alive = 0;

        int i = -1;
        loop (LOOP_NEIGHBORS) {
            int j = -1;

            loop (LOOP_NEIGHBORS) {
                if (!(i == 0 && j == 0)) {
                    if (oldBoard[GameOfLife.index(row + i, col + j)]) {
                        alive++;
                    }
                }
                j++;
            }

            i++;
        }

        int oldState = oldBoard[GameOfLife.index(row, col)];
        /* rule
        1. Any live cell with two or three live neighbours survives.
        2. Any dead cell with three live neighbours becomes a live cell.
        3. All other live cells die in the next generation. Similarly, all other dead cells stay dead.
        */
        return (alive == 3 || alive == 2 && oldState == LIVE) ? LIVE : DEAD;
    }

    static function index(int i, int j) : int {
        return i * GameOfLife.BOARD_SIZE + j;
    }

    function propagateState(SigHashPreimage txPreimage, int value) : bool {
        require(Tx.checkPreimage(txPreimage));
        bytes outputScript = this.getStateScript();
        bytes output = Utils.buildOutput(outputScript, value);
        return hash256(output) == SigHash.hashOutputs(txPreimage);
    }
}
```

