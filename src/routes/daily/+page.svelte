<script lang="ts">
  import GameInput from "$lib/components/GameInput.svelte";
  import GameFeed from "$lib/components/GameFeed.svelte";
  import Modal from "$lib/components/Modal.svelte";

  import { generateDiff } from "$lib/diff";
  import { sha256 } from "$lib/crypto";
  import { getDeltaDay } from "$lib/date";

  import data from "$lib/data.json";

  import { onMount } from "svelte";
  import toast from "svelte-french-toast";
  import pluralize from "pluralize";

  import { db } from "$lib/db";

  import { dailyStreak } from "$lib/stats";

  import LucideShare from "~icons/lucide/share";
  import { liveQuery } from "dexie";

  interface Country {
    code: string;
    name: string;
  }

  interface Guess extends Country {
    diff?: string;
    win?: boolean;
  }

  const dailyNumber = getDeltaDay("2024-06-06") * -1 + 1;

  const date = new Date().toLocaleDateString("en-US", {
    month: "numeric",
    day: "2-digit",
    year: "numeric",
  });

  let ISODate: string;

  // Game state
  let target: Country;
  let items: Guess[] = [];
  let isGameOver: boolean = false;

  let guesses: number = 0;

  let modal: Modal;

  const daily = liveQuery(() => db.daily.get(ISODate));

  onMount(async () => {
    // Generate a random number by hashing date and using the first 3 characters as a hex number
    const tzo = new Date().getTimezoneOffset() * 60_000;
    const date = new Date(Date.now() - tzo).toISOString().split("T")[0];
    ISODate = date;
    const hash = await sha256(date);
    const rnd = Number("0x" + hash.slice(0, 3)) / 16 ** 3;
    const index = Math.floor(rnd * data.length);
    target = data[index];

    // Fill previously guessed items
    const previouslyGuessed: Guess[] = JSON.parse(
      localStorage.getItem("daily-prev-guessed") || "[]",
    );
    const previouslyGuessedDate = localStorage.getItem("daily-prev-guessed-date");
    // Different day; reset
    if (!previouslyGuessedDate || previouslyGuessedDate != ISODate) {
      localStorage.removeItem("daily-prev-guessed");
    } else if (previouslyGuessed) {
      items = previouslyGuessed;
    }
  });

  // Show results if today has already been played
  daily.subscribe((daily) => {
    if (daily?.guesses) showResults();
  });

  async function addGuess(e: CustomEvent) {
    if (isGameOver) return;
    const country: Country = e.detail;
    const diff = await generateDiff(country, target);
    const win = checkWin(country);
    const guess: Guess = {
      code: country.code,
      name: country.name,
      diff: diff,
      win: win,
    };
    items = [guess, ...items];
    guesses++;
    if (win) {
      // Record today as win if there is not existing record
      const exists = (await db.daily.get(ISODate)) !== undefined;
      if (!exists) db.daily.put({ date: ISODate, guesses: items.length }, ISODate);
      // Show results modal
      modal.show();
    }
    // Store guess history
    localStorage.setItem("daily-prev-guessed", JSON.stringify(items));
    localStorage.setItem("daily-prev-guessed-date", ISODate);
  }

  function checkWin(guess: Country): boolean {
    if (target.code === guess.code) {
      isGameOver = true;
      return true;
    }
    return false;
  }

  function copyResults() {
    const resultString = `🏁 Flaggle #${dailyNumber} ${date} in ${pluralize("guess", $daily?.guesses || guesses, true)}! 👉 https://flaggle.kennyhui.dev/daily`;
    navigator.clipboard
      .writeText(resultString)
      .then(() => {
        toast.success("Copied results to clipboard");
      })
      .catch(() => {
        toast.error("Failed to copy to results clipboard");
      });
  }

  function showResults() {
    modal.show();
  }
</script>

<h1 class="font-[BigNoodleTitling] italic text-center text-4xl">
  Flaggle #{dailyNumber} <span class="text-base-content/50">{date}</span>
</h1>
{#if isGameOver || $daily?.guesses}
  <button
    class="font-[BigNoodleTitling] font-normal italic text-2xl btn self-center"
    on:click={showResults}>Results</button
  >
{:else}
  <GameInput on:submit={addGuess}></GameInput>
{/if}
<GameFeed {items}></GameFeed>

<Modal title="Results" bind:this={modal} centered>
  <p>You solved today's <b>Flaggle #{dailyNumber}</b> in</p>
  <p class="font-[BigNoodleTitling] italic text-5xl mb-2">
    {pluralize("guess", $daily?.guesses || guesses, true)}
  </p>
  <div class="flex gap-2">
    <button class="btn" on:click={copyResults}><LucideShare></LucideShare> Share Results</button>
  </div>
  <p>You now have a <b>{$dailyStreak} day</b> streak!</p>
</Modal>
