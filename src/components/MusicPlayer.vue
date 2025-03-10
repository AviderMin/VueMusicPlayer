<template>
  <div id="background" :style="{ backgroundImage: `url(${currentSong.cover})` }"></div>
  <div id="main-container">
    <div id="player-container">
      <div id="cover">
        <img :src="currentSong.cover" alt="Album Cover" />
      </div>
      <div id="song-info">
        <h2>{{ currentSong.title }}</h2>
        <p>{{ currentSong.artist }}</p>
      </div>
      <div id="progress-container">
        <span>{{ currentTime }}</span>
        <input type="range" v-model="progress" @input="seek" />
        <span>{{ duration }}</span>
      </div>
      <div id="controls">
        <button @click="prevSong">⏮</button>
        <button @click="togglePlay">{{ isPlaying ? '⏸' : '▶' }}</button>
        <button @click="nextSong">⏭</button>
      </div>
    </div>
    <ul id="lyrics-container" @click="toggleLyrics">
      <li v-for="(line, index) in lyrics" :key="index" :class="{ active: index === currentLyricIndex }">
        {{ line.text }}
      </li>
    </ul>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue';

const baseURL = 'http://avider.fun:5244/d/%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%AD%98%E5%82%A8/Cache/';
const audio = ref(new Audio());
const currentIndex = ref(0);
const isPlaying = ref(false);
const currentTime = ref('0:00');
const duration = ref('0:00');
const progress = ref(0);
const songs = ref([]);
const lyrics = ref([]);
const currentLyricIndex = ref(-1);

const currentSong = computed(() => songs.value[currentIndex.value] || {});

function getFullURL(path) {
  return new URL(path, baseURL).href;
}

async function loadSongs() {
  try {
    const response = await fetch('/songs.json');
    const data = await response.json();
    songs.value = data.map(song => ({
      ...song,
      src: getFullURL(song.src),
      cover: getFullURL(song.cover),
      lyricsSrc: getFullURL(song.lyricsSrc)
    }));
    loadSong();
  } catch (error) {
    console.error('加载歌曲列表失败:', error);
  }
}

async function loadLyrics() {
  try {
    const lyricsURL = getFullURL(currentSong.value.lyricsSrc);
    console.log('加载歌词 URL:', lyricsURL); // 调试信息
    const response = await fetch(lyricsURL);
    const text = await response.text();
    console.log('歌词内容:', text); // 调试信息
    lyrics.value = parseLyrics(text);
    console.log('解析后的歌词:', lyrics.value); // 调试信息
  } catch (error) {
    console.error('加载歌词失败:', error);
    lyrics.value = [];
  }
}
function parseLyrics(text) {
  const lines = text.split('\n');
  const regex = /\[(\d{2}):(\d{2})\.(\d{2,3})\](.*)/;
  return lines.map(line => {
    const match = line.match(regex);
    if (match) {
      const minutes = parseInt(match[1]);
      const seconds = parseInt(match[2]);
      const milliseconds = parseInt(match[3].padEnd(3, '0'));
      const time = minutes * 60 + seconds + milliseconds / 1000;
      return { time, text: match[4].trim() };
    }
    return null;
  }).filter(line => line !== null && line.text !== '');
}
function updateLyrics(currentTime) {
  if (!lyrics.value.length) return;
  let index = lyrics.value.findIndex(line => line.time > currentTime);
  index = index === -1 ? lyrics.value.length - 1 : index - 1;
  if (index !== currentLyricIndex.value) {
    currentLyricIndex.value = index;
    scrollToLyric(index);
  }
}

function scrollToLyric(index) {
  const container = document.getElementById('lyrics-container');
  const activeLine = container?.children[index];
  if (activeLine) {
    activeLine.scrollIntoView({
      behavior: 'smooth',
      block: 'center'
    });
  }
}

function loadSong() {
  if (songs.value.length === 0) return;
  audio.value.src = currentSong.value.src;
  audio.value.load();
  loadLyrics();
}

function togglePlay() {
  if (isPlaying.value) {
    audio.value.pause();
  } else {
    audio.value.play();
  }
  isPlaying.value = !isPlaying.value;
}

function prevSong() {
  currentIndex.value = (currentIndex.value - 1 + songs.value.length) % songs.value.length;
  loadSong();
  audio.value.play();
  isPlaying.value = true;
}

function nextSong() {
  currentIndex.value = (currentIndex.value + 1) % songs.value.length;
  loadSong();
  audio.value.play();
  isPlaying.value = true;
}

function seek() {
  audio.value.currentTime = (progress.value / 100) * audio.value.duration;
}

function formatTime(time) {
  const minutes = Math.floor(time / 60);
  const seconds = Math.floor(time % 60).toString().padStart(2, '0');
  return `${minutes}:${seconds}`;
}

onMounted(() => {
  loadSongs();
  audio.value.ontimeupdate = () => {
    progress.value = (audio.value.currentTime / audio.value.duration) * 100;
    currentTime.value = formatTime(audio.value.currentTime);
    duration.value = formatTime(audio.value.duration);
    updateLyrics(audio.value.currentTime);
  };
});

watch(currentIndex, loadLyrics);
</script>

<style scoped>


#background {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-size: cover;
  background-position: center;
  z-index: -1;
  transition: background-image 1s ease-in-out, filter 1s ease-in-out;
  filter: blur(200px);
  /* 应用模糊效果 */
  
  
  /* 毛玻璃效果 */
}

#main-container {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  width: 100%;
  max-width: 1200px;
  margin: auto; 
}

#player-container {
  padding: 30px;
  border-radius: 12px;
  text-align: center;
  width: 480px;
  max-width: 100%;
  box-shadow: 0 8px 12px rgba(0, 0, 0, 0.3);
  position: relative;
  z-index: 10;
}

#cover img {
  margin: 0 0 10px 0;
  width: 100%;
  border-radius: 12px;
  object-fit: cover;
}

#song-info h2 {
  margin: 10px 0;
  font-size: 1.6rem;
  font-weight: bold;
  color: #FFF;
}

#song-info p {
  font-size: 1.2rem;
  margin: 0;
  opacity: 0.8;
  color: #FFF;
}

#progress-container {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin: 20px 0;
}

#progress-container input[type="range"] {
  appearance: none;
  background-color: #fff !important;
  width: 70%;
  border-radius: 10px;
                       
}

#progress-container input[type="range"]::-webkit-slider-thumb {
  appearance: none;
  width: 15px;
  height: 15px;
  margin-top: -4px;
  border-radius: 50%;
  background-color: #fff;
}

input[type="range"]::-webkit-slider-runnable-track {
  height: 6px;
  background: linear-gradient(to right, #ffffff var(--progress), #fff var(--progress));
  border-radius: 5px;
}

#progress-bar {
  flex-grow: 1;
  margin: 0 15px;
  height: 6px;
  background-color: rgb(141, 141, 141);
  border-radius: 10px;
  cursor: pointer;
}

#controls button {
  font-size: 36px;
  background: none;
  border: none;
  color: white;
  cursor: pointer;
  margin: 0 15px;
  padding: 10px;
  transition: transform 0.2s ease;
  width: 10%; 
  text-align: center; 
}

#controls button:hover {
  transform: scale(1.2);
}

#lyrics-container {
  list-style: none;
  padding: 0 0 0 25px;
  max-height: 460px;
  width: 580px;
  overflow-y: auto;
  font-size: 32px;
  color: #fff;
  text-align: left;
  margin: auto;
  scroll-behavior: smooth;
  cursor: pointer;
}

#lyrics-container li {
  height: 45px;
  padding: 20px 0 0 0;
  opacity: 0.7;
  transition: opacity 0.6s, color 0.6s;
  white-space: nowrap;
  color: #fff;
  filter: blur(2px);
  font-weight: bold;
}

#lyrics-container li.active {
  color: #FFF;
  opacity: 1;
  font-weight: 900;
  filter: blur(0px);
}

#lyrics-container li.inactive {
  opacity: 0.6s;
}

#lyrics-container::-webkit-scrollbar {
  width: 6px;
  background: transparent;
}

#lyrics-container::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.4);
  border-radius: 10px;
}

@media (max-width: 600px) {
  #main-container {
    flex-direction: column;
    align-items: center;
  }

  #player-container {
    width: 100%;
    padding: 20px;
  }

  #cover img {
    width: 100%;
    max-width: 300px;
  }

  #song-info h2 {
    font-size: 1.4rem;
  }

  #song-info p {
    font-size: 1rem;
  }

  #controls button {
    font-size: 20px;
  }

  #lyrics-container {
    max-height: 300px;
    font-size: 1rem;
    width: 100%;
    margin: 20px 0 0 0;
  }
}
</style>