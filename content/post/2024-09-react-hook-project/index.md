---
title: "React Hook 介紹——透過踩地雷專案學 React"
description: 
date: 2024-09-08T22:56:57+02:00
image: 1.jpg
math: 
license: 
hidden: false
comments: true
draft: false
tags:
    - React
    - Hook
categories:
    - Modern Web
---


為了找工作，我開始自學練習寫 React，多虧了現今有 ChatGPT，可以很快的針對簡單的任務開發甚至是 debug。一邊寫、我也一邊看相關的教學，因此想統整這當中學到的內容寫成文章，專案最終的 [live demo](https://where-is-bijou.vercel.app/) 在這邊，不過目前只有很陽春的基本遊玩功能，希望可以找到時間把剩下的內容開發完成。

## 初始設定

如題所說，今天要來做一個踩地雷的 clone，想了很久 React 的第一份 project 應該要做什麼好，因為想要做可以互動的東西，結果想到我小時候很喜歡玩的遊戲…我真的很喜歡玩踩地雷，初級我曾經用 6 秒破關（到目前還是沾沾自喜的程度）。

廢話不多說，因為這篇文章主要想著墨在 React 的部分，所以專案設定我就簡單帶過～

```bash
npm create vite@latest minesweeper --template react
cd minesweeper
npm install

npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

使用 node 跟 npm 開啟一個 vite 新專案後，加上 tailwind CSS 作為其中一個 dependency，節省一些設計樣式的時間。

```jsx
// tailwind.config.js
module.exports = {
  content: [
    "./src/**/*.{js,jsx,ts,tsx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

```css
/** src/index.css **/
@tailwind base;
@tailwind components;
@tailwind utilities;
```

## 專案架構

### Component

整個網頁的架構可以用幾個 component 來說明邏輯跟資料流:

- Game - 創建遊戲、紀錄遊戲的邏輯、規則、輸贏
- Board - 遊戲盤的實體，裝有許多個 cell
- Cell - 每一個格子，紀錄了附近有幾個炸彈、被翻開了沒

（因為是第一次開發，所以沒有想得很清楚，請見諒）

Component 主要的想法就是把相同概念、領域、規範的程式碼寫在一起，與 class 的概念有點類似，透過寫出一個可以重複使用程式碼的方法，使得整體的程式好讀易懂。

## 初步開發

先把剛剛提到的架構元件寫出來

1. Cell

```jsx
// src/components/Cell.jsx

import React from 'react';

const Cell = ({ value, onClick, onContextMenu, row, col }) => {
  return (
    <div
      className="w-8 h-8 border flex items-center justify-center cursor-pointer"
      onClick={() => onClick(row, col)}
      onContextMenu={(e) => {
        e.preventDefault();
        onContextMenu(row, col);
      }}
    >
      {value}
    </div>
  );
};

export default Cell;
```

1. Board

```jsx
// src/components/Board.jsx

import React from 'react';
import Cell from './Cell';

const Board = ({ board, onClick, onContextMenu }) => {
  return (
    <div className="grid" style={{ gridTemplateColumns: `repeat(${board[0].length}, 2rem)` }}>
      {board.map((row, rowIndex) => 
        row.map((cell, colIndex) => 
          <Cell
            key={`${rowIndex}-${colIndex}`}
            value={cell.revealed ? (cell.value === 'M' ? '💣' : cell.value) : (cell.flagged ? '🚩' : '')}
            onClick={onClick}
            onContextMenu={onContextMenu}
            row={rowIndex}
            col={colIndex}
          />
        )
      )}
    </div>
  );
};

export default Board;
```

1. Game

`createBoard` 負責在一開始的時候生成棋盤、放置地雷、並且計算鄰居炸彈有幾個。

`handleCellClick` 處理如果點擊每個格子時的狀況，比如點到地雷則結束遊戲，如果不是地雷的話，則 `revealCells` 會把附近有幾個炸彈顯示出來。

<aside>
💡

阿突然想到，我在這邊放的不是地雷，是我家的貓咪——珠寶，所以以下的地雷、炸彈，我都改稱為珠寶 XD

</aside>

當使用右鍵點選格子的時候，`handleContextMenu` 會將這個標示成貓咪或是取消標示。

```jsx
// src/components/Game.jsx

import React, { useState, useEffect } from 'react';
import Board from './Board';

const createBoard = (rows, cols, mines) => {
  let board = Array(rows).fill().map(() => Array(cols).fill({ value: '', revealed: false, flagged: false }));
  let minePositions = [];

  while (minePositions.length < mines) {
    let row = Math.floor(Math.random() * rows);
    let col = Math.floor(Math.random() * cols);
    if (!minePositions.some(pos => pos[0] === row && pos[1] === col)) {
      minePositions.push([row, col]);
      board[row][col] = { ...board[row][col], value: 'M' };
    }
  }

  for (let row = 0; row < rows; row++) {
    for (let col = 0; col < cols; col++) {
      if (board[row][col].value !== 'M') {
        let minesCount = 0;
        for (let x = -1; x <= 1; x++) {
          for (let y = -1; y <= 1; y++) {
            if (board[row + x] && board[row + x][col + y] && board[row + x][col + y].value === 'M') {
              minesCount++;
            }
          }
        }
        if (minesCount > 0) board[row][col] = { ...board[row][col], value: minesCount };
      }
    }
  }

  return board;
};

const Game = () => {
  const [board, setBoard] = useState([]);
  const [gameOver, setGameOver] = useState(false);

  useEffect(() => {
    setBoard(createBoard(10, 10, 10));
  }, []);

  const handleCellClick = (row, col) => {
    if (gameOver) return;
    let newBoard = [...board];
    if (newBoard[row][col].value === 'M') {
      setGameOver(true);
      alert('Game Over');
    } else {
      revealCells(newBoard, row, col);
    }
    setBoard(newBoard);
  };

  const revealCells = (board, row, col) => {
    if (board[row][col].revealed || board[row][col].flagged) return;
    board[row][col].revealed = true;
    if (board[row][col].value === '') {
      for (let x = -1; x <= 1; x++) {
        for (let y = -1; y <= 1; y++) {
          if (board[row + x] && board[row + x][col + y]) {
            revealCells(board, row + x, col + y);
          }
        }
      }
    }
  };

  const handleContextMenu = (row, col) => {
    if (gameOver) return;
    let newBoard = [...board];
    newBoard[row][col].flagged = !newBoard[row][col].flagged;
    setBoard(newBoard);
  };

  return (
    <div className="flex justify-center items-center h-screen">
      <Board board={board} onClick={handleCellClick} onContextMenu={handleContextMenu} />
    </div>
  );
};

export default Game;

```

最後更新到 `<App>` component 當中

```jsx
// src/App.jsx

import React from 'react';
import Game from './components/Game';

const App = () => {
  return (
    <div className="App">
      <Game />
    </div>
  );
};

export default App;

```

### useState

在 `<Game>` 裡頭，使用`gameOver` 儲存遊戲是否結束了，這邊使用 useState 來監控特定變數，當這些變數有更動的時候，頁面才會部分重新 render。

useState 會以 array 的形式回傳兩個變數，這樣方便我們重新命名，通常第二個變數會等於第一個變數名稱加上 set，如這邊 `[gameOver, setGameOver]` 。其中 `setGameOver` 的參數可以帶入一個變數或是一個方法。

useState 也可以記錄 array 或是 class，像是`board`是一個 2D array，也可以透過 useState 監控。

### useEffect

```jsx
 useEffect(() => {
    setBoard(createBoard(10, 10, 10));
  }, []);
```

在這邊使用 useEffect，使得第一次的 render 會 call `createBoard()` 這個方法。

在 React 中，副作用邏輯（side effects logic）是指任何不直接影響 UI render，但與外部環境或系統交互的操作。例如，call API、訂閱或事件監聽、計時器操作（如設置 `setTimeout` 或 `setInterval）` 、local storage 或 session 的操作。這些操作通常需要與應用外部的資源或狀態進行交互，因此稱為「副作用」。使用 `useEffect` 來執行 side effects logic，可以讓 React 在 component 的生命週期中正確處理這些邏輯，如在 mount、update、或 unmount。

useEffect 的第二個參數是一個 array，列舉了依賴哪些項目，也就是說當這個 array 裡的東西有了變動，useEffect 內的邏輯就會被呼叫。總共有三種情況:

1. array 為空 `[]` - 只有在第一次 render 的時候會呼叫
2. array 有塞值進去 - 內容改變時會呼叫
3. 不傳 array 進去 - 每次 render 的時候都會呼叫（這個情形的話形同虛設、沒有提升效率）

## 功能——產生不同的棋盤大小

架構寫好了之後，來增加實際玩的機制
首先，我們想根據不同的**難度**，來產生不同大小的棋盤

```jsx
// src/components/Board.jsx

import React, { useMemo } from 'react';
import Cell from './Cell';

const generateTable = () => {
    return Array.from({ length: rows }).map(() =>
        Array.from({ length: cols }).map(() => Math.floor(Math.random() * 10)) // Random number between 0 and 9
    );
};
    
const Board = ({ difficulty }) => {
    const { rows, cols } = useMemo(() => {
        switch (difficulty) {
            case 'easy':
                return { rows: 5, cols: 5 };
            case 'medium':
                return { rows: 10, cols: 10 };
            case 'hard':
                return { rows: 15, cols: 15 };
            default:
                return { rows: 10, cols: 10 };
        }
    }, [difficulty]);

    const table = useMemo(generateTable, [rows, cols]);

    return (
        <>
            {table.map((row, rowIndex) => (
                <div key={rowIndex} className="flex flex-row justify-center">
                    {row.map((cell, colIndex) => (
                        <Cell key={colIndex} neighborMine={cell} />
                    ))}
                </div>
            ))}
        </>
    );
}

export default Board;

```

### useMemo

當遇到複雜的計算時，為了避免重複計算，useMemo 將計算的結果記起來。像是這邊把`rows`跟 `cols` 記起來，然後當每次 `difficulty` 有更動的時候，則會重新運算、並將結果存起來。接著 `table` 也是一樣使用 useMemo 每次 `[rows, cols]` 有產生變化時，就會把新生成的棋盤存起來。

因為這樣， `<Game>` 和 `<Cell>` 都可以簡化了！

```jsx
// src/components/Game.jsx

import React, { useState } from 'react';
import Board from './Board';

export const Game = () => {
    const [difficulty, setDifficulty] = useState('medium');

    return (
        <div className="container my-10 mx-auto py-8 max-w-4xl border border-pink-400">
            <label className="flex flex-row justify-center">
                Difficulty:
                <select name="difficulty" value={difficulty} onChange={e => setDifficulty(e.target.value)}>
                    <option value="easy">Easy</option>
                    <option value="medium">Medium</option>
                    <option value="hard">Hard</option>
                </select>
            </label>
            <br />
            <Board difficulty={difficulty} />
        </div>
    );
}
```

```jsx
// src/components/Cell.jsx

import React from 'react';

export const Cell = ({ neighborMine }) => {
    return (
        <button className="cell bg-transparent hover:bg-blue-500 text-blue-700 font-semibold 
            hover:text-white py-0 px-1 border border-blue-200 hover:border-transparent">
            {neighborMine}
        </button>
    );
}

export default Cell;

```

## 功能——連帶打開沒有珠寶的格子的鄰居

當我點某個格子，其中的值是 0 的時候，代表這一個不是炸… 貓咪、鄰居也都不是貓咪，所以理當應該把鄰居們也都遞迴打開。

```jsx
// src/component/Board.jsx

import React, { useState, useMemo, useCallback } from 'react';
import Cell from './Cell';

const Board = ({ difficulty }) => {
    const { rows, cols, mines } = useMemo(() => {
        switch (difficulty) {
            case 'easy':
                return { rows: 5, cols: 5, mines: 5 };
            case 'medium':
                return { rows: 10, cols: 10, mines: 20 };
            case 'hard':
                return { rows: 15, cols: 15, mines: 45 };
            default:
                return { rows: 10, cols: 10, mines: 20 };
        }
    }, [difficulty]);

    const generateTable = (mines) => {
        const table = Array.from({ length: rows }, () =>
            Array.from({ length: cols }, () => 0)
        );

        let placedMines = 0;
        while (placedMines < mines) {
            const row = Math.floor(Math.random() * rows);
            const col = Math.floor(Math.random() * cols);

            if (table[row][col] !== -1) {
                table[row][col] = -1;
                placedMines++;
            }
        }

        return table;
    };

    const table = useMemo(() => generateTable(mines), [rows, cols, mines]);
    const [statuses, setStatuses] = useState(
        Array.from({ length: rows }, () => Array.from({ length: cols }, () => 'unknown'))
    );

    const revealCells = useCallback((row, col, newStatuses, table) => {
        if (row < 0 || row >= rows || col < 0 || col >= cols || newStatuses[row][col] === 'revealed') {
            return;
        }

        newStatuses[row][col] = 'revealed';
        if (table[row][col] === 0) {
            revealCells(row - 1, col - 1, newStatuses, table);
            revealCells(row - 1, col, newStatuses, table);
            revealCells(row - 1, col + 1, newStatuses, table);
            revealCells(row, col - 1, newStatuses, table);
            revealCells(row, col + 1, newStatuses, table);
            revealCells(row + 1, col - 1, newStatuses, table);
            revealCells(row + 1, col, newStatuses, table);
            revealCells(row + 1, col + 1, newStatuses, table);
        }
    }, [rows, cols]);

    const handleCellClick = useCallback((row, col, clickType) => {
        setStatuses(prevStatuses => {
            const newStatuses = prevStatuses.map(rowStatus => [...rowStatus]);

            if (clickType === 'left') {
                if (table[row][col] === 0) {
                    revealCells(row, col, newStatuses, table);
                } else {
                    newStatuses[row][col] = 'revealed';
                }
            } else if (clickType === 'right') {
                newStatuses[row][col] = newStatuses[row][col] === 'flagged' ? 'unknown' : 'flagged';
            }

            return newStatuses;
        });
    }, [revealCells, table]);

    return (
        <>
            {table.map((row, rowIndex) => (
                <div key={rowIndex} className="flex flex-row justify-center">
                    {row.map((cell, colIndex) => (
                        <Cell
                            key={colIndex}
                            neighborMine={cell}
                            status={statuses[rowIndex][colIndex]}
                            onClick={(clickType) => handleCellClick(rowIndex, colIndex, clickType)}
                        />
                    ))}
                </div>
            ))}
        </>
    );
};

export default Board
```

### useCallback

*useCallback 的主要目的是避免在 component 內部宣告的 function，因為隨著每次 render 不斷重新被宣告跟建立，每次拿到的都是不同的 instance。這樣的 function 如果被當成 prop 往下傳給其他 component，就可能導致下面的 component 無意義地被重新 render。*

*但是除非你的 component 有實作比對 prop 做選擇性 render 的檢查，不然就算傳了 useCallback 記下來的 function 進去也毫無意義——他的 render function 還不是會被跑一次。考慮到這一點的話：**大部分情況下，我們都不需要用到** useCallback。*

*如果你的 function 因為需要用到 props 或 state 而必須在 component scope 裡面宣告、但又同時會被超過一個 useEffect 使用時，就建議以 useCallback 包起來。*（[解釋得非常清楚，原文在此](https://medium.com/ichef/%E4%BB%80%E9%BA%BC%E6%99%82%E5%80%99%E8%A9%B2%E4%BD%BF%E7%94%A8-usememo-%E8%B7%9F-usecallback-a3c1cd0eb520)）

我觀察的情況是， `handleCellClick` 的確有符合使用一些 states，所以必須包在 component 裡頭，接著在棋盤上會不斷的被 call，因此看起來這是一個合理的 practice！不過 `handleCellClick` 裡頭又藏了一個 useCallback function `revealCells` ，看起來實在有點複雜。

```jsx
// src/components/Cell.jsx

import React from 'react';

export const Cell = ({ neighborMine, status, onClick }) => {
    const handleClick = (e) => {
        e.preventDefault(); // Prevent context menu on right-click
        if (e.nativeEvent.button === 0) {
            onClick('left'); // Handle left click
        } else if (e.nativeEvent.button === 2) {
            onClick('right'); // Handle right click
        }
    };

    return (
        <>
            {status === 'unknown' ? (
                <div
                    className="cell hover:bg-blue-500 hover:text-blue-500 bg-blue-100 
                    text-blue-100 py-0 px-2 border border-blue-400 font-mono cursor-pointer"
                    onContextMenu={handleClick}
                    onClick={handleClick}
                >
                    '
                </div>
            ) : status === 'flagged' ? (
                <div
                    className="cell bg-yellow-500 text-white py-0 px-2 border border-yellow-400 font-mono cursor-pointer"
                    onContextMenu={handleClick}
                    onClick={handleClick}
                >
                    F
                </div>
            ) : (
                <button className="cell bg-transparent text-blue-700 font-semibold 
                    py-0 px-2 border border-blue-400 font-mono">
                    {neighborMine}
                </button>
            )}
        </>
    );
};

export default Cell;

```

然後到了這邊，遊戲的狀態有 `unknown` 、 `flagged` 、 `unreveal` ，會呈現相對應的格子。

## UI——新增剩餘的珠寶

套用一樣的概念，我們用變數 `remainingMines` 搭配 useState 來監控剩下的珠寶。

```jsx
// src/components/Game.jsx

import React, { useState } from 'react';
import Board from './Board';

export const Game = () => {
    const [difficulty, setDifficulty] = useState('medium');
    const [remainingMines, setRemainingMines] = useState(20); // Initial value for medium difficulty

    const handleDifficultyChange = (e) => {
        const newDifficulty = e.target.value;
        setDifficulty(newDifficulty);
        switch (newDifficulty) {
            case 'easy':
                setRemainingMines(5);
                break;
            case 'medium':
                setRemainingMines(20);
                break;
            case 'hard':
                setRemainingMines(45);
                break;
            default:
                setRemainingMines(20);
        }
    };

    return (
        <>
            <div className="container my-10 mx-auto py-8 max-w-4xl border border-pink-400">
                <label className="flex flex-row justify-center">
                    Difficulty:
                    <select name="difficulty" value={difficulty} onChange={handleDifficultyChange}>
                        <option value="easy">Easy</option>
                        <option value="medium">Medium</option>
                        <option value="hard">Hard</option>
                    </select>
                </label>
                <br />
                <div className="flex flex-row justify-center">
                    <span>Mines left: {remainingMines}</span>
                </div>
                <Board difficulty={difficulty} setRemainingMines={setRemainingMines} />
            </div>
        </>
    );
}
```

```jsx
// src/component/Board.jsx

import React, { useState, useMemo, useCallback, useEffect } from 'react';
import Cell from './Cell';

const Board = ({ difficulty, setRemainingMines }) => {
    const { rows, cols, mines } = useMemo(() => {
        switch (difficulty) {
            case 'easy':
                return { rows: 5, cols: 5, mines: 5 };
            case 'medium':
                return { rows: 10, cols: 10, mines: 20 };
            case 'hard':
                return { rows: 15, cols: 15, mines: 45 };
            default:
                return { rows: 10, cols: 10, mines: 20 };
        }
    }, [difficulty]);

    const generateTable = (mines) => {
        const table = Array.from({ length: rows }, () =>
            Array.from({ length: cols }, () => 0)
        );

        const statuses = Array.from({ length: rows }, () =>
            Array.from({ length: cols }, () => 'unknown')
        );

        let placedMines = 0;
        while (placedMines < mines) {
            const row = Math.floor(Math.random() * rows);
            const col = Math.floor(Math.random() * cols);

            if (table[row][col] !== -1) {
                table[row][col] = -1;
                placedMines++;
            }
        }

        return { table, statuses };
    };

    const { table, initialStatuses } = useMemo(() => generateTable(mines), [rows, cols, mines]);
    const [statuses, setStatuses] = useState(initialStatuses);

    useEffect(() => {
        setStatuses(Array.from({ length: rows }, () => Array.from({ length: cols }, () => 'unknown')));
    }, [difficulty, rows, cols]);

    const revealCells = useCallback((row, col, newStatuses, table) => {
        const stack = [[row, col]];

        while (stack.length) {
            const [r, c] = stack.pop();

            if (r < 0 || r >= rows || c < 0 || c >= cols || newStatuses[r][c] === 'revealed') {
                continue;
            }

            newStatuses[r][c] = 'revealed';

            if (table[r][c] === 0) {
                stack.push([r - 1, c - 1]);
                stack.push([r - 1, c]);
                stack.push([r - 1, c + 1]);
                stack.push([r, c - 1]);
                stack.push([r, c + 1]);
                stack.push([r + 1, c - 1]);
                stack.push([r + 1, c]);
                stack.push([r + 1, c + 1]);
            }
        }
    }, [rows, cols]);

    const handleCellClick = useCallback((row, col, clickType) => {
        setStatuses(prevStatuses => {
            const newStatuses = prevStatuses.map(rowStatus => [...rowStatus]);

            if (clickType === 'left') {
                if (table[row][col] === 0) {
                    revealCells(row, col, newStatuses, table);
                } else {
                    newStatuses[row][col] = 'revealed';
                }
            } else if (clickType === 'right') {
                if (newStatuses[row][col] === 'flagged') {
                    newStatuses[row][col] = 'unknown';
                    setRemainingMines(prev => prev + 1);
                } else {
                    newStatuses[row][col] = 'flagged';
                    setRemainingMines(prev => prev - 1);
                }
            }

            return newStatuses;
        });
    }, [revealCells, table, setRemainingMines]);

    return (
        <>
            {table.map((row, rowIndex) => (
                <div key={rowIndex} className="flex flex-row justify-center">
                    {row.map((cell, colIndex) => (
                        <Cell
                            key={colIndex}
                            neighborMine={cell}
                            status={statuses[rowIndex][colIndex]}
                            onClick={(clickType) => handleCellClick(rowIndex, colIndex, clickType)}
                        />
                    ))}
                </div>
            ))}
        </>
    );
};

export default Board;

```

```jsx
// src/component.jsx

import React from 'react';

const Cell = ({ neighborMine, status, onClick }) => {
    const handleClick = (e) => {
        e.preventDefault();
        if (e.type === 'click' && e.button === 0) {
            onClick('left');
        } else if (e.type === 'contextmenu' && e.button === 2) {
            onClick('right');
        }
    };

    return (
        <>
            {status === 'revealed' ? (
                <button 
                    className="cell bg-transparent text-blue-700 font-semibold py-0 px-2 border border-blue-400 font-mono"
                    onClick={handleClick}
                    onContextMenu={handleClick}
                >
                    {neighborMine === 0 ? <>&nbsp;</> : neighborMine}
                </button>
            ) : (
                <div 
                    className="cell hover:bg-blue-500 hover:text-blue-500 bg-blue-100 text-blue-100 py-0 px-2 border border-blue-400 font-mono"
                    onClick={handleClick}
                    onContextMenu={handleClick}
                >
                    &nbsp;
                </div>
            )}
        </>
    );
};

export default Cell;

```

## Debug

在來回修改程式的過程中，我常常得到一個 warning 

> fix warning: Warning: Cannot update a component (`Game`) while rendering a different component (`Board`).
> 

我研究了一下，發現這是因為在嘗試 render child component 的時候試著更新 parent component，所以可以透過 

1. 將 state update logic 移動到 parent component
2. 更新 child component ，並確保 child component 只透過 callback 來跟 parent component 溝通，而不是直接更新

來解決，也就是 useEffect、useCallback 等 hook 發揮用處的時候到了！

## UI——新增遊戲結局

玩到結束，終究是要顯示輸贏！這邊我想使用一個彈跳視窗，所以需要新增一個 `Modal` component

```jsx
// src/components/Modal.jsx

import React from 'react';

const Modal = ({ message, onClose }) => {
    return (
        <div className="fixed inset-0 flex items-center justify-center z-50">
            <div className="bg-white p-4 rounded shadow-lg">
                <div className="mb-4">{message}</div>
                <button
                    className="bg-blue-500 text-white px-4 py-2 rounded"
                    onClick={onClose}
                >
                    Close
                </button>
            </div>
            <div className="fixed inset-0 bg-gray-500 opacity-50"></div>
        </div>
    );
};

export default Modal;

```

同時，我也新增顯示時間正計時、狀態的顯示 (playing、won、lost) 。

```jsx
// src/components/Game.jsx

import React, { useState, useEffect, useCallback } from 'react';
import Board from './Board';
import Modal from './Modal';

export const Game = () => {
    const [difficulty, setDifficulty] = useState('medium');
    const [remainingMines, setRemainingMines] = useState(20);
    const [gameStatus, setGameStatus] = useState('prepare');
    const [startTime, setStartTime] = useState(null);
    const [elapsedTime, setElapsedTime] = useState(0);
    const [modalMessage, setModalMessage] = useState('');

    const handleDifficultyChange = (e) => {
        const newDifficulty = e.target.value;
        setDifficulty(newDifficulty);
    };

    const updateGameStatus = useCallback((unrevealedCount) => {
        if (unrevealedCount === remainingMines) {
            setGameStatus('won');
        }
    }, [remainingMines]);

    useEffect(() => {
        switch (difficulty) {
            case 'easy':
                setRemainingMines(5);
                break;
            case 'medium':
                setRemainingMines(20);
                break;
            case 'hard':
                setRemainingMines(45);
                break;
            default:
                setRemainingMines(20);
        }
        setGameStatus('prepare');
        setStartTime(null);
        setElapsedTime(0);
    }, [difficulty]);

    useEffect(() => {
        let timer;
        if (gameStatus === 'playing') {
            timer = setInterval(() => {
                setElapsedTime(prevTime => prevTime + 1);
            }, 1000);
        } else {
            clearInterval(timer);
        }

        return () => clearInterval(timer);
    }, [gameStatus]);

    const handleCellClick = useCallback((clickType, neighborMine) => {
        if (clickType === 'left' && gameStatus === 'prepare') {
            setGameStatus('playing');
            setStartTime(Date.now());
        }

        if (clickType === 'left' && neighborMine === '*') {
            setGameStatus('lost');
        }
    }, [gameStatus]);

    useEffect(() => {
        if (gameStatus === 'won') {
            setModalMessage('You won :)');
        } else if (gameStatus === 'lost') {
            setModalMessage('You lost :(');
        }
    }, [gameStatus]);

    const closeModal = () => {
        setModalMessage('');
    };

    return (
        <div className="container my-10 mx-auto py-8 max-w-4xl border border-pink-400">
            <label className="flex flex-row justify-center">
                Difficulty:
                <select name="difficulty" value={difficulty} onChange={handleDifficultyChange}>
                    <option value="easy">Easy</option>
                    <option value="medium">Medium</option>
                    <option value="hard">Hard</option>
                </select>
            </label>
            <br />
            <div className="flex flex-row justify-center">
                <span>Mines left: {remainingMines}</span>
                <span className="ml-4">Status: {gameStatus}</span>
                <span className="ml-4">Time: {elapsedTime}s</span>
            </div>
            <Board
                difficulty={difficulty}
                setRemainingMines={setRemainingMines}
                updateGameStatus={updateGameStatus}
                setGameStatus={setGameStatus}
                handleCellClick={handleCellClick}
            />
            {modalMessage && <Modal message={modalMessage} onClose={closeModal} />}
        </div>
    );
};

```

- 新增了 `modalMessage` 來儲存要在 Modal 中顯示的訊息。
- 更新監視 `gameStatus` 的 useEffect hook，以在遊戲獲勝或失敗時設定 `modalMessage`

## 部署

最後還修修改改了許多東西，不過主要與 React 有關的內容已經大致涵蓋了！因此如果想看完整程式碼的話，可以直接參考 [source code](https://github.com/Miyaya/where-is-bijou)。

我用的是 [Vercel](https://vercel.com/miyayas-projects)，一個免費部署、CI/CD 的 server 服務，使用起來非常簡單，因為可以直接從 GitHub 導入專案，以及許多 framework 的編譯模板，像是 vite 就有包含在內，因此只要一開始設定的時候勾選是 vite 就好了。

## 結論

最後，如果我的理解有誤、或是想要提供更好的作法，歡迎留言！也可以寫信給我 miya850604[at]gmail.com，謝謝^^