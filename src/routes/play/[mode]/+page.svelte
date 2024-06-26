<script lang="ts">
  import type { PageData } from "./$types";
	import { onMount } from "svelte";
  import { slide } from "svelte/transition";
  import { Separator } from "bits-ui";
  import {
    NumberFormatter,
    type Movie,
    type AnswerState,
    localScore,
    localUsername,
  } from "$lib/utils";

  import MovieInfoDialog from "$lib/components/MovieInfoDialog.svelte";
  import MoviePoster from "$lib/components/MoviePoster.svelte";
  import Button from "$lib/components/Button.svelte";
  import Tooltip from "$lib/components/Tooltip.svelte";
  import Leaderboard from "$lib/components/Leaderboard.svelte";

  export let data: PageData;

  const animationDuration = 500;

  let movie1: Movie;
  let movie2: Movie;
  let movieNext: Movie;
  let movieInfo: Movie;

  let round = 1;
  let state: "load" | "game" | "end" = "load";
  let answerState: AnswerState = null;
  let infoDialogOpen = false;
  let highScore = false;
  let leaderboardOpen = false;

  async function fetchMovie() {
    const response = await fetch("/api/random-movie", {
      method: "GET",
      headers: {
        "Content-Type": "application/json"
      }
    });

    return await response.json();
  }

  async function setMovies() {
    movie1 = await fetchMovie();
    movie2 = await fetchMovie();
    while (checkSameStat(movie1, movie2)) movie2 = await fetchMovie();
    movieNext = await fetchMovie();
    while (checkSameStat(movie2, movieNext)) movieNext = await fetchMovie();

    state = "game";
  }

  function guessPick(value: "higher" | "lower") {
    if (answerState) return;

    let movie1value: number;
    let movie2value: number;

    switch (data.gameMode) {
      case "budget":
        movie1value = movie1.budget;
        movie2value = movie2.budget;
        break;
      case "rating":
        movie1value = movie1.rating;
        movie2value = movie2.rating;
        break;
      case "time":
        movie1value = new Date(movie1.release_date).getTime();
        movie2value = new Date(movie2.release_date).getTime();
        break;
    }

    if (value === "higher") {
      if (movie2value > movie1value) correct();
      else incorrect();
    } else {
      if (movie2value < movie1value) correct();
      else incorrect();
    }
  }

  async function correct() {
    round++;
    answerState = "correct";

    let movieNextPrev = movieNext;
    movieNext = await fetchMovie();
    while (checkSameStat(movie2, movieNext)) movieNext = await fetchMovie();

    setTimeout(async () => {
      movie1 = movie2;
      movie2 = movieNextPrev;

      answerState = null;
    }, animationDuration * 2);
  }

  function incorrect() {
    answerState = "incorrect";
    if (round > (Number(localScore(data.gameMode)) || 0)) {
      updateScore();
      highScore = true;
    }

    setTimeout(() => {
      state = "end";
    }, animationDuration * 2);
  }

  function setMovieInfo(movie: Movie) {
    infoDialogOpen = true;
    movieInfo = movie;
  }

  function checkSameStat(movie1: Movie, movie2: Movie) {
    switch (data.gameMode) {
      case "budget": return movie1.budget === movie2.budget;
      case "rating": return movie1.rating === movie2.rating;
      case "time": return new Date(movie1.release_date).getFullYear() === new Date(movie2.release_date).getFullYear();
    }
  }

  async function updateScore() {
    if (!localUsername()) return;

    localScore(data.gameMode, round);
    const response = await fetch("/api/leaderboard", {
      method: "POST",
      body: JSON.stringify({
        score: round,
        mode: data.gameMode,
        username: localUsername()
      }),
      headers: {
        "Content-Type": "application/json",
      },
    });
  }

  onMount(setMovies);
</script>

<!-- Loading movies -->
{#if state === "load"}
  <main class="h-full bg-imdb flex justify-center items-center" transition:slide>
    <h1 class="text-5xl font-impactt flex md:flex-row flex-col items-center md:gap-2 gap-8">
      LOADING MOVIES...
      <iconify-icon icon="humbleicons:spinner-earring" class="animate-spin"></iconify-icon>
    </h1>
  </main>

<!-- Game over -->
{:else if state === "end"}
  <main class="h-full bg-black flex flex-col justify-center items-center gap-8" transition:slide>
    <h1 class="bg-imdb px-2.5 py-1 rounded-md text-black font-impactt text-5xl">GAME OVER!</h1>
    <h2 class="text-2xl">
      You made it to
      <span class="border-2 border-white rounded-lg px-1.5 py-0.5 ">ROUND {round}</span>
    </h2>
    {#if highScore}
      <h2 class="text-imdb text-2xl">You beat your high score!</h2>
    {/if}
    <div class="flex justify-center items-center md:gap-6 gap-4">
      <!-- Home -->
      <a
        href="/"
        class="flex justify-center items-center size-24 bg-imdb rounded-3xl duration-150 hover:scale-110 active:scale-95">
        <iconify-icon
          icon="mingcute:home-7-fill"
          class="text-6xl text-white"
        ></iconify-icon>
      </a>

      <!-- Play again -->
      <button
        on:click={() => location.reload()}
        class="flex justify-center items-center size-32 bg-imdb rounded-3xl duration-150 hover:scale-110 active:scale-95">
        <iconify-icon
          icon="mingcute:refresh-4-fill"
          class="text-6xl text-white duration-200 group-hover:rotate-180"
        ></iconify-icon>
      </button>

      <!-- Leaderboard -->
      <button
        on:click={() => (leaderboardOpen = true)}
        class="flex justify-center items-center size-24 bg-imdb rounded-3xl duration-150 hover:scale-110 active:scale-95">
        <iconify-icon
          icon="material-symbols:trophy-rounded"
          class="text-6xl text-white duration-200"
        ></iconify-icon>
      </button>
    </div>
  </main>

<!-- Game -->
{:else if state === "game"}
  <!-- Desktop version -->
  <main class="md:flex hidden h-full w-full justify-between relative" transition:slide>
    <!-- Home button -->
    <a
      href="/"
      class="text-white bg-transparent border-2 border-white rounded-full flex justify-end items-end size-32 absolute -left-16 -top-16 duration-150 hover:bg-white z-50 hover:text-black"
    >
      <iconify-icon icon="mingcute:home-7-fill" class="text-2xl mr-6 mb-6"></iconify-icon>
    </a>

    <!-- Poster 1 -->
    <MoviePoster
      movie={movie1}
      animationDuration={animationDuration}
      state={answerState}
      infoButtonClick={() => setMovieInfo(movie1)}
    />

    <div class="flex flex-col items-center justify-evenly flex-grow">
      <!-- First movie info -->
      <div class="flex flex-col items-center gap-2 justify-center">
        <!-- Movie title -->
        <button
          on:click={() => setMovieInfo(movie1)}
          class="border-2 border-white rounded-lg px-3 py-1 font-impactt flex items-center justify-center text-center gap-2 mx-4 text-2xl hover:scale-[104%] active:scale-100 duration-150"
        >
          <iconify-icon icon="material-symbols:arrow-circle-left-rounded"></iconify-icon>
          <span>{movie1.title}</span>
        </button>

        <!-- Budget mode -->
        {#if data.gameMode === "budget"}
          <h5 class="text-lg">has a budget of</h5>
          <h2 class="bg-imdb rounded-md px-3 py-1 text-black font-impactt text-5xl">
            ${NumberFormatter.format(movie1.budget)}
          </h2>

        <!-- Rating mode -->
        {:else if data.gameMode === "rating"}
          <h5 class="text-lg">has a rating of</h5>
          <Tooltip text={movie1.rating.toString()}>
            <h2 class="bg-imdb rounded-md px-3 py-1 text-black font-impactt flex items-center text-3xl w-full h-full">
              <div class="relative w-full h-full">
                <div
                  class="absolute h-full text-clip whitespace-nowrap overflow-hidden"
                  style:width="{movie1.rating / 10 * 300}px"
                >
                  {#each [0,1,2,3,4,5,6,7,8,9] as i}
                    <iconify-icon icon="mingcute:star-fill" class="absolute" style:left="{i * 30}px"></iconify-icon>
                  {/each}
                </div>
              </div>
              <div class="flex w-full h-full justify-center items-center">
                {#each [0,1,2,3,4,5,6,7,8,9] as _}
                  <iconify-icon icon="mingcute:star-line"></iconify-icon>
                {/each}
              </div>
            </h2>
          </Tooltip>

        {:else if data.gameMode === "time"}
          <h5 class="text-lg">was released in the year</h5>
          <h2 class="bg-imdb rounded-md px-3 py-1 text-black font-impactt text-5xl">
            {new Date(movie1.release_date).getFullYear()}
          </h2>
        {/if}
      </div>

      <div class="flex gap-4 items-center justify-center w-full px-8">
        <Separator.Root class="h-[2px] flex-grow bg-white" />
        <!-- <iconify-icon icon="bxs:camera-movie" class="text-3xl -scale-x-100"></iconify-icon> -->
        <h1 class="text-3xl font-medium font-impactt">ROUND {round}</h1>
        <!-- <iconify-icon icon="bxs:camera-movie" class="text-3xl"></iconify-icon> -->
        <Separator.Root class="h-[2px] flex-grow bg-white" />
      </div>

      <!-- Second movie -->
      <div class="flex flex-col items-center gap-2 justify-center w-full">
        <!-- Movie title -->
        <button
          on:click={() => setMovieInfo(movie2)}
          class="border-2 border-white rounded-lg px-3 py-1 font-impactt flex items-center justify-center text-center gap-2 mx-4 text-2xl hover:scale-[104%] active:scale-100 duration-150"
        >
          <span>{movie2.title}</span>
          <iconify-icon icon="material-symbols:arrow-circle-right-rounded"></iconify-icon>
        </button>

        <!-- Budget mode -->
        {#if data.gameMode === "budget"}
          <h5 class="text-lg">has a</h5>

          <div class="space-y-2 w-full px-8">
            <Button on:click={() => guessPick("higher")} icon="material-symbols:arrow-upward-alt-rounded">Higher</Button>
            <Button on:click={() => guessPick("lower")} icon="material-symbols:arrow-downward-alt-rounded">Lower</Button>
          </div>

          <h5 class="text-lg">budget</h5>

        <!-- Rating mode -->
        {:else if data.gameMode === "rating"}
          <h5 class="text-lg">has a</h5>

          <div class="space-y-2 w-full px-8">
            <Button on:click={() => guessPick("higher")} icon="material-symbols:arrow-upward-alt-rounded">Higher</Button>
            <Button on:click={() => guessPick("lower")} icon="material-symbols:arrow-downward-alt-rounded">Lower</Button>
          </div>

          <h5 class="text-lg">rating</h5>

        <!-- Time mode -->
        {:else if data.gameMode === "time"}
          <h5 class="text-lg">is</h5>

          <div class="space-y-2 w-full px-8">
            <Button on:click={() => guessPick("higher")} icon="material-symbols:arrow-upward-alt-rounded">Newer</Button>
            <Button on:click={() => guessPick("lower")} icon="material-symbols:arrow-downward-alt-rounded">Older</Button>
          </div>
        {/if}
      </div>
    </div>

    <!-- Poster 2 -->
    <MoviePoster
      movie={movie2}
      animationDuration={animationDuration}
      state={answerState}
      infoButtonClick={() => setMovieInfo(movie2)}
    />
  </main>

  <!-- Mobile version -->
  <main class="flex flex-col md:hidden h-full w-full justify-between relative" transition:slide>
    <!-- Posters -->
    <div class="flex w-full relative">
      <!-- Answer state -->
      {#if answerState}
        <div
          class="absolute w-full h-full bg-imdb"
          transition:slide={{ duration: animationDuration, axis: "y" }}
        >
          {#if answerState === "correct"}
            <div class="flex flex-col justify-center items-center gap-2 h-full">
              <iconify-icon icon="mingcute:check-fill" class="text-6xl text-black"></iconify-icon>
              <h1 class="text-5xl text-black font-impactt">Correct</h1>
            </div>
          {:else if answerState === "incorrect"}
            <div class="flex flex-col justify-center items-center gap-2 h-full">
              <iconify-icon icon="mingcute:close-fill" class="text-6xl text-black"></iconify-icon>
              <h1 class="text-5xl text-black font-impactt">Incorrect</h1>
            </div>
          {/if}
        </div>
      {/if}

      <!-- Home button -->
      <a
        href="/"
        class="text-white bg-transparent border-2 border-white rounded-full flex justify-center items-center size-20 absolute left-1/2 -translate-x-1/2 -top-10 duration-150 hover:bg-white z-50 hover:text-black"
      >
        <iconify-icon icon="mingcute:home-7-fill" class="text-xl mt-8"></iconify-icon>
      </a>

      <button
        on:click={() => setMovieInfo(movie1)}
        style:background-image="url('https://image.tmdb.org/t/p/original{movie1.poster_path}')" class="w-1/2 bg-cover bg-center duration-150 group-hover:scale-[105%] aspect-[2/3] h-min max-h-[25rem]"
      ></button>
      <button
        on:click={() => setMovieInfo(movie2)}
        style:background-image="url('https://image.tmdb.org/t/p/original{movie2.poster_path}')" class="w-1/2 bg-cover bg-center duration-150 group-hover:scale-[105%] aspect-[2/3] h-min max-h-[25rem]"
      ></button>
    </div>

    <div class="flex-grow flex justify-center items-center">
      <div class="flex flex-col items-center justify-evenly flex-grow h-full">
        <!-- First movie info -->
        <div class="flex flex-col items-center gap-2 justify-center">
          <!-- Movie title -->
          <!-- <button
            on:click={() => setMovieInfo(movie1)}
            class="border-2 border-white rounded-lg px-3 py-1 font-impactt flex items-center justify-center text-center gap-2 mx-4 text-2xl hover:scale-[104%] active:scale-100 duration-150"
          >
            <iconify-icon icon="material-symbols:arrow-circle-left-rounded"></iconify-icon>
            <span>{movie1.title}</span>
          </button> -->

          <!-- Budget mode -->
          {#if data.gameMode === "budget"}
            <h5 class="text-xl flex items-center gap-1">
              <iconify-icon icon="material-symbols:arrow-circle-left-rounded"></iconify-icon>
              has a budget of
            </h5>
            <h2 class="bg-imdb rounded-md px-3 py-1 text-black font-impactt text-4xl">
              ${NumberFormatter.format(movie1.budget)}
            </h2>

          <!-- Rating mode -->
          {:else if data.gameMode === "rating"}
            <h5 class="text-xl flex items-center gap-1">
              <iconify-icon icon="material-symbols:arrow-circle-left-rounded"></iconify-icon>
              has a rating of
            </h5>
            <Tooltip text={movie1.rating.toString()}>
              <h2 class="bg-imdb rounded-md px-3 py-1 text-black font-impactt flex items-center text-3xl w-full h-full">
                <div class="relative w-full h-full">
                  <div
                    class="absolute h-full text-clip whitespace-nowrap overflow-hidden"
                    style:width="{movie1.rating / 10 * 300}px"
                  >
                    {#each [0,1,2,3,4,5,6,7,8,9] as i}
                      <iconify-icon icon="mingcute:star-fill" class="absolute" style:left="{i * 30}px"></iconify-icon>
                    {/each}
                  </div>
                </div>
                <div class="flex w-full h-full justify-center items-center">
                  {#each [0,1,2,3,4,5,6,7,8,9] as _}
                    <iconify-icon icon="mingcute:star-line"></iconify-icon>
                  {/each}
                </div>
              </h2>
            </Tooltip>

          {:else if data.gameMode === "time"}
            <h5 class="text-xl flex items-center gap-1">
              <iconify-icon icon="material-symbols:arrow-circle-left-rounded"></iconify-icon>
              was released in the year
            </h5>
            <h2 class="bg-imdb rounded-md px-3 py-1 text-black font-impactt text-5xl">
              {new Date(movie1.release_date).getFullYear()}
            </h2>
          {/if}
        </div>

        <div class="flex gap-4 items-center justify-center w-full px-8 my-2">
          <Separator.Root class="h-[2px] flex-grow bg-white" />
          <h1 class="text-3xl font-medium font-impactt">ROUND {round}</h1>
          <Separator.Root class="h-[2px] flex-grow bg-white" />
        </div>

        <!-- Second movie -->
        <div class="flex flex-col items-center gap-2 justify-center w-full">
          <!-- Movie title -->
          <!-- <button
            on:click={() => setMovieInfo(movie2)}
            class="border-2 border-white rounded-lg px-3 py-1 font-impactt flex items-center justify-center text-center gap-2 mx-4 text-2xl hover:scale-[104%] active:scale-100 duration-150"
          >
            <span>{movie2.title}</span>
            <iconify-icon icon="material-symbols:arrow-circle-right-rounded"></iconify-icon>
          </button> -->
          <!-- <button
            on:click={() => setMovieInfo(movie1)}
            class="font-impactt flex items-center text-center gap-2 mx-4 text-2xl"
          >
            <span>{movie2.title}</span>
            <iconify-icon icon="material-symbols:arrow-circle-right-rounded"></iconify-icon>
          </button> -->

          <!-- Budget mode -->
          {#if data.gameMode === "budget"}
            <h5 class="text-xl flex items-center gap-1">
              <iconify-icon icon="material-symbols:arrow-circle-right-rounded"></iconify-icon>
              has a
            </h5>

            <div class="space-y-2 w-full px-8">
              <Button on:click={() => guessPick("higher")} icon="material-symbols:arrow-upward-alt-rounded">Higher</Button>
              <Button on:click={() => guessPick("lower")} icon="material-symbols:arrow-downward-alt-rounded">Lower</Button>
            </div>

            <h5 class="text-lg">budget</h5>

          <!-- Rating mode -->
          {:else if data.gameMode === "rating"}
            <h5 class="text-xl flex items-center gap-1">
              <iconify-icon icon="material-symbols:arrow-circle-right-rounded"></iconify-icon>
              has a
            </h5>

            <div class="space-y-2 w-full px-8">
              <Button on:click={() => guessPick("higher")} icon="material-symbols:arrow-upward-alt-rounded">Higher</Button>
              <Button on:click={() => guessPick("lower")} icon="material-symbols:arrow-downward-alt-rounded">Lower</Button>
            </div>

            <h5 class="text-lg">rating</h5>

          <!-- Time mode -->
          {:else if data.gameMode === "time"}
            <h5 class="text-xl flex items-center gap-1">
              <iconify-icon icon="material-symbols:arrow-circle-right-rounded"></iconify-icon>
              was released
            </h5>

            <div class="space-y-2 w-full px-8">
              <Button on:click={() => guessPick("higher")} icon="material-symbols:arrow-upward-alt-rounded">Later</Button>
              <Button on:click={() => guessPick("lower")} icon="material-symbols:arrow-downward-alt-rounded">Earlier</Button>
            </div>
          {/if}
        </div>
      </div>
    </div>
  </main>
{/if}

<MovieInfoDialog
  bind:open={infoDialogOpen}
  movie={movieInfo}
  gameMode={data.gameMode}
/>

<Leaderboard bind:open={leaderboardOpen} />
