<script>
  import { onMount } from "svelte";
  import axios from "axios";
  import Canvas from "./Canvas.svelte";
  import Spinner from "./Spinner.svelte";
  import Modal from "./Modal.svelte";
  import { scrollPlayEnabled } from "./stores.js";

  let canvasesAvailable,
    canvases = [];
  let allAudio = [];
  let loading = true;
  let mobile,
    canvasLimitReached,
    scrollPlayLoopBusy,
    showModal,
    audioInitialized = false;

  const MOBILE_WIDTH = 414;
  const NUM_CANVASES_MOBILE = 5;
  const SCREEN_HEIGHT = window.innerHeight;

  if (window.innerWidth <= MOBILE_WIDTH) {
    mobile = true;
  }

  onMount(async () => {
    if (mobile) {
      setInterval(scrollPlayLoop, 20);
    }
    canvasesAvailable = await getCanvases();
    loading = false;

    if (mobile && canvasesAvailable.length > NUM_CANVASES_MOBILE) {
      canvases = canvasesAvailable.slice(0, NUM_CANVASES_MOBILE);
    } else {
      canvases = canvasesAvailable;
      canvasLimitReached = true;
    }

    allAudio = canvases.map(canvas => new Audio(canvas.previewUrl));
    console.log("allAudio:", allAudio);

    document.body.addEventListener("touchstart", tapped, false);
  });

  function tapped() {
    if (!audioInitialized && allAudio) {
      console.log("audio init started", new Date());
      for (let audio of allAudio) {
        audio.play();
        audio.pause();
        audio.currentTime = 0;
      }
      setTimeout(function() {
        allAudio[0].play();
      }, 10);
      audioInitialized = true;
      console.log("audio init finished", new Date());
    }
  }

  function scrollPlayLoop() {
    if (scrollPlayLoopBusy) return;
    if (!$scrollPlayEnabled) return;
    if (showModal) return;
    scrollPlayLoopBusy = true;

    let bestIndex = -1;
    let bestPixVisible = 0;
    let canvasDivs = document.querySelectorAll(".canvas");

    if (window.scrollY > 0) {
      canvasDivs.forEach((div, i) => {
        let divHeight = div.offsetHeight;
        let divRect = div.getBoundingClientRect();
        let divTop = divRect.top;
        let divBot = divRect.bottom;

        if (divTop > SCREEN_HEIGHT) divTop = SCREEN_HEIGHT;
        else if (divTop < 0) divTop = 0;

        if (divBot >= SCREEN_HEIGHT) divBot = SCREEN_HEIGHT;
        else if (divBot < 0) divBot = 0;

        let pixVisible = divBot - divTop;
        if (pixVisible > bestPixVisible) {
          bestPixVisible = pixVisible;
          bestIndex = i;
        }
      });
    }
    if (bestIndex !== -1) {
      let audioElement = allAudio[bestIndex];
      if (audioElement && audioElement.paused) {
        playSound(audioElement, bestIndex);
      }
    }
    scrollPlayLoopBusy = false;
  }

  function playSound(audioElement, index) {
    for (let i = 0; i < allAudio.length; i++) {
      if (i === index) continue;
      allAudio[i].pause();
    }
    audioElement.volume = 0.5;
    let promise = audioElement.play();
    if (promise) {
      promise
        .then(_ => {
          console.log("autoplay started for", index);
        })
        .catch(catchId => {
          console.log("autoplay prevented for", index, "catchId:", catchId);
          showModal = true;
        });
    }
  }

  function getCanvases() {
    return axios
      .get("/api/canvases")
      .then(response => {
        if (response.data.error) {
          console.log("ERROR /api/canvases:", response.data.error);
        } else {
          return response.data.canvases;
        }
      })
      .catch(error => console.log("ERROR /api/canvases:", error));
  }

  function loadMore() {
    const maxCanvases = canvasesAvailable.length;
    const numDisplayed = canvases.length;
    const newNumDisplayed = numDisplayed + NUM_CANVASES_MOBILE;
    let canvasesAdded;

    if (newNumDisplayed < maxCanvases) {
      canvases = canvasesAvailable.slice(0, newNumDisplayed);
      canvasesAdded = canvasesAvailable.slice(numDisplayed, newNumDisplayed);
    } else {
      canvases = canvasesAvailable.slice(0, maxCanvases);
      canvasesAdded = canvasesAvailable.slice(numDisplayed, maxCanvases);
      canvasLimitReached = true;
    }
    canvasesAdded.map(canvas => allAudio.push(new Audio(canvas.previewUrl)));
    for (let i = numDisplayed; i < newNumDisplayed; i++) {
      allAudio[i].play();
      allAudio[i].pause();
      allAudio[i].currentTime = 0;
    }
  }
</script>

<h1>spotify <span class="nowrap">canvas</span></h1>
<h3 class="yellow">my recently played songs with canvases updated automatically</h3>
<p class="pink">hover over, scroll, or touch the videos to play audio</p>
<div class="container">
  {#if loading}
    <Spinner/>
  {/if}
  {#if canvases.length > 0}
    {#each canvases as canvas, index}
      <Canvas
        {canvas}
        {index}
        {mobile}
        audioElement={allAudio[index]}
        {playSound}
        on:playblocked={() => showModal = true}
      />
    {/each}
    {#if !loading && !canvasLimitReached}
      <button on:click={loadMore} class="button">more</button>
    {/if}
  {:else if !loading && canvases.length === 0}
    <h3>No canvases found ☹️</h3>
  {/if}
</div>
<p class="footer">
  <a href="https://umarhadi.dev" target="_blank" rel="noopener noreferrer">
    made with&nbsp;
    <span role="img" aria-label="red heart">❤️</span>
    by umarhadi
  </a>
</p>
{#if showModal}
	<Modal on:close="{() => {
    showModal = false
  }}"/>
{/if}

<style>
  .nowrap {
    white-space: nowrap;
  }
  .green {
    color: #4aff68;
  }
  .pink {
    color: #fb65b2;
  }
  .yellow {
    color: #feff6e;
    margin-bottom: 30px;
  }
  .blue {
    color: #72ffe4;
  }
  .container {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
  }
  .footer {
    font-size: 12px;
    margin-top: 30px;
  }
  .button {
    margin-top: 10px;
    font-size: 18px;
    line-height: 1;
    border-radius: 500px;
    padding: 21px 40px 19px;

    color: #fff;
    background-color: #1db954;
    border-color: #1aa34a;
    transition-property: background-color, border-color, color, box-shadow, filter;
    transition-duration: 0.3s;
    border-width: 0;
    letter-spacing: 2px;
    min-width: 160px;
    white-space: normal;

    display: inline-block;
    margin-bottom: 0;
    font-weight: 700;
    text-align: center;
    vertical-align: middle;
    touch-action: manipulation;
    cursor: pointer;
    background-image: none;
    border: 1px solid transparent;
    user-select: none;
  }
  .button:hover {
    background-color: #1ed760;
    border-color: #116e32;
    text-decoration: none;
  }
  .button:active {
    background-color: #1aa34a;
    border-color: #116e32;
    box-shadow: none;
  }
</style>
