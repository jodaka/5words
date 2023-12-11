<script lang="ts">
  import { dict as WORDS } from './dict';
  //https://github.com/Harrix/Russian-Nouns/blob/main/dist/russian_nouns.txt
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
    withoutLetters: Set<string>;
    withLetters: Set<string>;
  }

  const getWordsSuggestions = (
    dict: Dict,
    { positionsPositive, positionsNegative, withoutLetters, withLetters }: ILimits,
    count = 25
  ) => {
    if (!withLetters.size && !withoutLetters.size) {
      return [];
    }

    const res: string[] = [];

    let i = 0;
    mainLoop: while (i < dict.length && res.length < count) {
      const word = dict[i];
      i++;

      const letters: string[] = word.split('');
      const lettersSet = new Set(letters);

      // проверка на обязательные буквы
      for (const withLetter of withLetters) {
        if (!lettersSet.has(withLetter)) {
          continue mainLoop;
        }
      }

      for (const withoutLetter of withoutLetters) {
        if (lettersSet.has(withoutLetter)) {
          continue mainLoop;
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
            continue mainLoop;
          }
        }
      }

      // проверка на отсутствие букв по указанным позициям
      const positiveLetters = Object.keys(positionsPositive);
      for (let i = positiveLetters.length - 1; i >= 0; i--) {
        const positiveLetter = positiveLetters[i];
        const rawValue = positionsPositive[positiveLetter];
        const positiveLetterPositions = Array.isArray(rawValue)
          ? rawValue
          : typeof rawValue === 'number'
          ? [rawValue]
          : Array.from(rawValue);

        for (let positivePosIndex = positiveLetterPositions.length - 1; positivePosIndex >= 0; positivePosIndex--) {
          const positivePos = positiveLetterPositions[positivePosIndex];
          if (letters[positivePos - 1] !== positiveLetter) {
            continue mainLoop;
          }
        }
      }

      res.push(word);
    }

    return res;
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
  let gridValues: IGridCell[][] = new Array(6).fill(null).map(() =>
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

  const toggleState = (gridCell: IGridCell, toggleOnlyEmptyState: boolean = false): void => {
    if (gridCell.value === '') {
      gridCell.state = GRID_STATES.empty;
    } else if (toggleOnlyEmptyState) {
      gridCell.state = GRID_STATES.absent;
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
      toggleState(gridCell, true);
      return;
    }

    if (cellIndex < 24) {
      evt.preventDefault();
      gridCell.value = evt.key;
      focusInput(cellIndex + 1);
      toggleState(gridCell, true);
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
              if (!positionsPositive.hasOwnProperty(letter.value)) {
                positionsPositive[letter.value] = new Set();
              }
              positionsPositive[letter.value].add(letterIndex + 1);
              break;

            case GRID_STATES.exists:
              withLetters.add(letter.value);

              if (!positionsNegative.hasOwnProperty(letter.value)) {
                positionsNegative[letter.value] = new Set();
              }
              positionsNegative[letter.value].add(letterIndex + 1);
              break;
          }
        });
      }
    });

    const res: ILimits = {
      withLetters,
      withoutLetters,
      positionsNegative,
      positionsPositive
    };
    return res;
  };

  let limits: any = {};
  let wordsSuggestions: string[] = [];

  $: limits = calculateLimits(gridValues);
  $: wordsSuggestions = getWordsSuggestions(WORDS, limits, 25);

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

  {#if wordsSuggestions.length}
    <ul class="suggestions">
      <h3>Варианты варианты:</h3>
      {#each wordsSuggestions as suggestion}
        <li>{suggestion}</li>
      {/each}
    </ul>
  {/if}

  {#if initialSuggestions.length}
    <ul class="suggestions">
      <h3>Начальные варианты:</h3>
      {#each initialSuggestions as suggestion}
        <li>{suggestion}</li>
      {/each}
    </ul>
  {/if}
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
    grid-template-columns: min-content min-content;
    grid-template-rows: min-content;
    margin: 0 auto;

    justify-content: center;
  }

  .suggestions {
    color: var(--letter-color);
    padding: 0 1rem;
    margin: 0;
    list-style: none;
  }

  .suggestions h3 {
    margin-top: 0;
  }

  .suggestions li {
    font-family: monospace;
    letter-spacing: 0.1rem;
    text-transform: uppercase;
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
    gap: 10px;
    width: fit-content;
    height: fit-content;
  }

  * {
    box-sizing: border-box;
  }

  @media only screen and (min-device-width: 320px) and (max-device-width: 480px) {
  }
</style>
