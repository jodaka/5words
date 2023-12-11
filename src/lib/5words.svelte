<script lang="ts">
  import { dict as WORDS } from './dict';

  type Dict = string[];

  enum GRID_STATES {
    'empty',
    'absent',
    'inplace',
    'exists'
  }

  interface IGridCell {
    value: string;
    state: GRID_STATES;
  }

  type LetterLimit = Record<string, number[]> | Record<string, number> | Record<string, Set<number>>;

  interface ILimits {
    positionsPositive: LetterLimit;
    positionsNegative: LetterLimit;
    withoutLetters: string[];
    withLetters: string[];
  }

  const getWordsSuggestions = (
    dict: Dict,
    { positionsPositive = {}, positionsNegative = {}, withoutLetters = [], withLetters = [] }: ILimits
  ) => {
    const withoutLettersSet = new Set(withoutLetters);

    return dict.filter((word) => {
      const letters: string[] = word.split('');

      // проверка на обязательные буквы
      for (let i = 0; i < withLetters.length; i++) {
        if (!letters.includes(withLetters[i])) {
          return false;
        }
      }

      // проверка на запрещённые буквы
      for (let i: number = 0; i <= 4; i++) {
        if (withoutLettersSet.has(letters[i])) {
          return false;
        }
      }

      // проверка на отсутствие букв по указанным позициям
      const negativeLetters = Object.keys(positionsNegative);
      for (let i = negativeLetters.length - 1; i >= 0; i--) {
        const negativeLetter = negativeLetters[i];
        const rawValue = positionsNegative[negativeLetter];
        const negativeLetterPositions = Array.isArray(rawValue)
          ? rawValue
          : typeof rawValue === 'number'
          ? [rawValue]
          : Array.from(rawValue);

        for (let negativePosIndex = negativeLetterPositions.length - 1; negativePosIndex >= 0; negativePosIndex--) {
          const negativePos = negativeLetterPositions[negativePosIndex];
          if (letters[negativePos - 1] === negativeLetter) {
            return false;
          }
        }
      }

      return true;
    });
  };

  const getInitialSuggestions = (dict: Dict = [], count = 10) => {
    const res: string[] = [];
    let i = 0;

    while (res.length < count && i < dict.length) {
      const word = dict[i];

      const letters = new Set(word.split(''));
      if (letters.size === 5) {
        res.push(word);
      }
      i++;
    }

    return res;
  };

  // Начальное состояние
  let gridValues: IGridCell[][] = new Array(5).fill(null).map(() =>
    new Array(5).fill(null).map(
      (): IGridCell => ({
        state: GRID_STATES.empty,
        value: ''
      })
    )
  );

  const focusInput = (inputIndex: number): void => {
    const el: HTMLInputElement | null = document.querySelector(`#cell_${inputIndex}`);
    if (el) {
      el.focus();
      return;
    }
  };

  const toggleState = (gridCell: IGridCell): void => {
    if (gridCell.value === '') {
      gridCell.state = GRID_STATES.empty;
    } else {
      switch (gridCell.state) {
        case GRID_STATES.empty:
          gridCell.state = GRID_STATES.absent;
          break;
        case GRID_STATES.absent:
          gridCell.state = GRID_STATES.exists;
          break;
        case GRID_STATES.exists:
          gridCell.state = GRID_STATES.inplace;
          break;
        case GRID_STATES.inplace:
          gridCell.state = GRID_STATES.empty;
          break;
      }
    }
    gridValues = gridValues;
  };

  const onKeyDown = (evt: KeyboardEvent, gridCell: IGridCell, cellIndex: number): void => {
    console.log(evt.code, cellIndex, gridCell);

    if (evt.code === 'ArrowLeft') {
      if (cellIndex > 0) {
        focusInput(cellIndex - 1);
      }
      return;
    }

    if (evt.code === 'ArrowRight') {
      if (cellIndex < 24) {
        focusInput(cellIndex + 1);
      }
      return;
    }

    if (evt.code === 'Backspace') {
      evt.preventDefault();

      if (gridCell.value !== '') {
        gridCell.value = '';
      }
      if (cellIndex > 0) {
        focusInput(cellIndex - 1);
      }
      toggleState(gridCell);
      return;
    }

    if (cellIndex < 24) {
      evt.preventDefault();
      gridCell.value = evt.key;
      focusInput(cellIndex + 1);
      toggleState(gridCell);
      return;
    }
  };

  const calculateLimits = (gridValues: IGridCell[][]): ILimits => {
    let withLetters: Set<string> = new Set();
    let withoutLetters: Set<string> = new Set();
    let positionsNegative: Record<string, Set<number>> = {};
    let positionsPositive: Record<string, Set<number>> = {};

    gridValues.forEach((line) => {
      const word = line.map(({ value }) => value).join('');

      if (word.length === 5) {
        line.forEach((letter: IGridCell, letterIndex: number) => {
          switch (letter.state) {
            case GRID_STATES.absent:
              withoutLetters.add(letter.value);
              break;
            case GRID_STATES.inplace:
              withLetters.add(letter.value);
              if (!positionsPositive[letter.value]) {
                positionsPositive[letter.value] = new Set();
              }
              positionsPositive[letter.value].add(letterIndex + 1);
              break;
              break;
            case GRID_STATES.exists:
              withLetters.add(letter.value);

              if (!positionsNegative[letter.value]) {
                positionsNegative[letter.value] = new Set();
              }
              positionsNegative[letter.value].add(letterIndex + 1);
              break;
          }
        });
      }
    });

    return {
      withLetters: Array.from(withLetters),
      withoutLetters: Array.from(withoutLetters),
      positionsNegative,
      positionsPositive
    };
  };

  let limits: any = {};
  let wordsSuggestions: string[] = [];

  $: limits = calculateLimits(gridValues);
  $: wordsSuggestions = getWordsSuggestions(WORDS, limits);

  let initialSuggestions = getInitialSuggestions(WORDS);
</script>

<main class="wrapper">
  <div class="grid">
    {#each gridValues as gridWord, wordIndex (wordIndex)}
      {#each gridWord as gridCell, cellIndex (cellIndex)}
        <input
          on:click={() => toggleState(gridCell)}
          on:keydown={(evt) => onKeyDown(evt, gridCell, wordIndex * 5 + cellIndex)}
          class="gridInput"
          id={`cell_${wordIndex * 5 + cellIndex}`}
          class:gridInput--absent={gridCell.state === GRID_STATES.absent ||
            (gridCell.state === GRID_STATES.empty && gridCell.value !== '')}
          class:gridInput--inplace={gridCell.state === GRID_STATES.inplace}
          class:gridInput--exists={gridCell.state === GRID_STATES.exists}
          type="text"
          maxlength="1"
          bind:value={gridCell.value}
        />
      {/each}
    {/each}
  </div>

  <div class="debug">
    {JSON.stringify(gridValues, null, 2)}
  </div>

  <ul class="suggestions">
    {JSON.stringify(limits, null, 2)}
    {JSON.stringify(wordsSuggestions)}
  </ul>

  <ul class="suggestions">
    {#each initialSuggestions as suggestion}
      <li>{suggestion}</li>
    {/each}
  </ul>
</main>

<style>
  .wrapper {
    background-color: #282830;
    min-height: 100dvh;

    --color-white: #fff;
    --color-yellow: #ffdd2d;
    --color-yellow-hover: #ffcd33;
    --color-yellow-active: #fab619;
    --color-gray: #9299a2;
    --app-background: #1c1c1e;
    --modal-background: #2c2c2e;
    --dark-button-color: #2c2c2e;
    --letter-color: #fff;
    --letter-correct-color: #000;
    --letter-present-color: #000;
    --letter-absent-color: #fff;
    --leter-border-color: var(--color-yellow);
    --letter-correct-border-color: var(--color-yellow);
    --letter-present-border-color: #fff;
    --letter-absent-border-color: #5f5f5f;
    --letter-correct-background: var(--color-yellow);
    --letter-present-background: #fff;
    --letter-absent-background: #5f5f5f;

    display: grid;
    grid-template-columns: 3fr 1fr 1fr;
  }

  .suggestions {
    /* display: flex; */
    color: var(--letter-color);
    padding: 1rem;
  }

  .debug {
    white-space: pre;
    border: 1px solid red;
    overflow: scroll;
    color: var(--letter-color);
  }

  .gridInput {
    width: 62px;
    height: 66px;
    text-align: center;
    border: 1px solid var(--color-yellow);
    border-radius: 6px;
    background-color: transparent;
    outline: none;
    color: white;
    font-size: 1.6rem;
    user-select: none;
    text-transform: capitalize;
  }

  .gridInput--absent {
    background: var(--letter-absent-background);
    border-color: var(--letter-absent-border-color);
    color: var(--letter-absent-color);
  }

  .gridInput--inplace {
    background: var(--letter-correct-background);
    border-color: var(--letter-correct-border-color);
    color: var(--letter-correct-color);
  }

  .gridInput--exists {
    background: var(--letter-present-background);
    border-color: var(--letter-present-border-color);
    color: var(--letter-present-color);
  }

  .grid {
    margin: 0 auto;
    display: grid;
    grid-template-columns: 1fr 1fr 1fr 1fr 1fr;
    max-width: 600px;
    gap: 10px;
    width: fit-content;
    height: fit-content;
    user-select: none;
  }

  * {
    box-sizing: border-box;
  }

  @media only screen and (min-device-width: 320px) and (max-device-width: 480px) {
  }
</style>
