<template lang="html">
    <div class="container">
        <div
          v-on:dragover.prevent="dragOverHandler"
          v-on:drop.prevent="dropHandler"
          class="player">
          <header class="player__header">
            <img
              src="img/logo.svg"
              alt="logo"
              class="player__logo">
            <h1 class="player__heading">{{ heading }}</h1>
          </header>
          <!-- <ul class="player__list" v-if="list.length > 0"> -->
          <ul class="player__list">
            <li
              draggable="true"
              v-bind:class="{
                'player__list__i': true,
                'dragging': item.dragging,
                'over': item.over,
                'current': item.isCurrent,
                'playing': item.isPlaying,
                'paused': item.isPaused
                }"
              v-on:click="songSelectHandler(idx)"
              v-on:dragstart="dragItemStartHandler"
              v-on:dragend="dragItemEndHandler"
              v-on:dragover.prevent="dragOverItemHandler"
              v-on:dragleave="dragLeaveItemHandler"
              v-on:drop="dropItemHandler"
              v-bind:data-id="idx"
              v-for="item, idx in list">{{ item.name }}</li>
          </ul>
          <!-- <div
            v-if="list.length > 0"
            class="player__progressbar"> -->
          <div
            class="player__progressbar">
            <div
              v-on:click="progressBarHandler"
              class="player__progressbar__scale">
              <div
                v-bind:style="{width: `${progress}%`}"
                class="player__progressbar__scale__current"></div>
            </div>
            <div
              class="player__progressbar__time"
              v-if="audio">{{currentTime}}</div>
          </div>
          <div class="player__ctrls">
            <button
              type="button"
              class="player__ctrl"
              v-html="btn.code"
              v-on:click="ctrlHandler(btn.name)"
              v-for="btn in btns"></button>
          </div>
        </div>
    </div>
</template>
<script>
    export default {
        name: 'App',
        mounted() {
          window.addEventListener('keydown', this.keyDownHandler);
        },
        data() {
            return {
                heading: 'Gramophone',
                btns: [
                  {
                    name: 'play',
                    code: '&#9654;'
                  },
                  {
                    name: 'pause',
                    code: '&#10074;&#10074;'
                  },
                  {
                    name: 'prev',
                    code: '&#10096;'
                  },
                  {
                    name: 'next',
                      code: '&#10097;'
                  }
                ],
                list: [],
                currentId: 0,
                audio: null,
                progress: 0,
                currentTime: '00:00',
                draggingItemIdx: null
            };
        },
        methods: {
          keyDownHandler(evt) {
            let audio = this.audio;
            if (evt.code === 'Space' && this.audio && audio.currentTime > 0) {
              evt.preventDefault();
              audio.paused ? this.ctrlHandler('play') : this.ctrlHandler('pause');
            } else {
              return false;
            }
          },
          songSelectHandler(idx) {
            this.setSong(idx);
          },
          ctrlHandler(btn) {
            if (!this.list.length) {
              return false;
            }

            switch(true) {
              case btn === 'play':
                this.handleSongState(btn);
                break;
              case btn === 'pause':
                this.handleSongState(btn);
                break;
              case btn === 'prev':
                let prevId = this.currentId - 1;

                if (this.currentId === 0) {
                  prevId = this.list.length - 1;
                }

                this.setSong(prevId, true);
                break;
              case btn === 'next':
                let nextId = this.currentId + 1;

                if (this.currentId === this.list.length - 1) {
                  nextId = 0;
                }

                this.setSong(nextId, true);
                break;
            }
          },
          setSong(songId, play) {
            this.handleSongsFlags(this.resetSongsFlags);
            this.handleSongsFlags(this.toggleSongFlags('isCurrent', songId));
            this.currentId = songId;

            if (this.audio && this.audio.currentTime > 0 && !this.audio.paused) {
              this.audio.pause();
            }
            this.audio = new Audio(this.list[songId].src);
            this.setAudioEvents();

            if (play) {
              this.audio.play();
            }
          },
          handleSongState(state) {
            let song = this.list[this.currentId];

            if (!this.audio) {
              return false;
            }

            this.audio[state]();
            song.isPaused = state === 'pause' ? true : false;
            song.isPlaying = state === 'play' ? true : false;
          },
          setAudioEvents() {
            this.audio.addEventListener('timeupdate', this.timeupdateHandler);
          },
          timeupdateHandler(evt) {
            let currentTime = Math.round(this.audio.currentTime);

            this.progress = this.audio.currentTime / (this.audio.duration / 100);
            this.currentTime = this.parseTime(currentTime);
          },
          parseTime(rawTime) {
            let parsed;

            if (rawTime > 60) {
              let minutes = Math.floor(rawTime / 60),
                remainder = rawTime % 60,
                seconds = remainder < 10 ? `0${remainder}` : remainder;

              parsed = `0${minutes}:${seconds}`;
            } else if (rawTime < 10) {
              parsed = `00:0${rawTime}`;
            } else {
              parsed = `00:${rawTime}`;
            }

            return parsed;
          },
          toggleSongFlags(flag, songId) {
            return (item, idx) => {
              if (idx === songId) {
                item[flag] = !item[flag];
              } else if (item[flag]) {
                item[flag] = false;
              }
            }
          },
          resetSongsFlags(item) {
            if (item.isCurrent) {
              item.isCurrent = false;
            }
            if (item.isPaused) {
              item.isPaused = false;
            }
            if (item.over) {
              item.over = false;
            }
            if (item.dragging) {
              item.dragging = false;
            }
          },
          handleSongsFlags(cb) {
            this.list.forEach(cb);
          },
          progressBarHandler(evt) {
            if (this.audio && evt.offsetX) {
              let timePercetage = evt.offsetX / (evt.currentTarget.clientWidth / 100),
                time = timePercetage * (this.audio.duration / 100);

              this.audio.currentTime = time;
            }
          },
          parseName(fullName) {
            let parsed = fullName.split('.');

            parsed = parsed.slice(0, parsed.length - 1).join('');

            return parsed;
          },
          dropHandler(evt) {
            Array.prototype.slice.call(evt.dataTransfer.items).forEach((item) => {
              let file = item.getAsFile(),
                reader = new FileReader();

              reader.addEventListener('load', (evt) => {
                this.list.push({
                  name: this.parseName(file.name),
                  src: reader.result,
                  dragging: false,
                  over: false,
                  isPlaying: false,
                  isCurrent: false,
                  isPaused: false
                });
              });
              reader.readAsDataURL(file);

              if (file.type !== 'audio/mp3') {
                throw new Error('Unsupported file format!');
              }
            });
          },
          dragItemStartHandler(evt) {
            evt.dataTransfer.effectAllowed = 'move';
            this.draggingItemIdx = parseInt(evt.currentTarget.dataset.id, 10);

            this.list[this.draggingItemIdx].dragging = true;
          },

          dragItemEndHandler(evt) {
            this.list[this.draggingItemIdx].dragging = false;
          },

          dragOverItemHandler(evt) {
            this.toggleOverItemHandler(evt, true);
          },
          dragLeaveItemHandler(evt) {
            this.toggleOverItemHandler(evt, false);
          },
          toggleOverItemHandler(evt, over) {
            let targetIdx = parseInt(evt.currentTarget.dataset.id, 10),
              draggingItemIdx = this.draggingItemIdx;

            this.handleItems(((item, idx) => {
              if (targetIdx !== draggingItemIdx && targetIdx === idx) {
                item.over = over;
              }
            }));
          },
          handleItems(cb) {
            this.list.forEach(cb);
          },
          dropItemHandler(evt) {
            let targetIdx = parseInt(evt.currentTarget.dataset.id, 10),
              draggingItemIdx = this.draggingItemIdx,
              list = this.list.slice(),
              temp = list[targetIdx];

            list[targetIdx] = list[draggingItemIdx];
            list[draggingItemIdx] = temp;
            this.list = list;
            this.handleSongsFlags(this.resetSongsFlags);
          },
          dragOverHandler(evt) {}
        }
    };
</script>
<style lang="less">
  html,
  body {
    margin: 0;
    padding: 0;
  }
</style>
<style lang="less" scoped>
    @colorBlue: rgb(44, 168, 208);
    @colorLightBlue: rgb(76, 192, 230);
    @colorGrey: rgb(91, 87, 83);
    @colorWhite: rgb(255, 255, 255);
    @colorYellow: rgb(238, 201, 87);

    .borderRadius(@radius: 4px) {
      border-radius: @radius;
    }

    @keyframes blink {
      0% {
        background: @colorGrey;

      }
      100% {
        background: @colorLightBlue;
      }
    }
    .container {
        // width: 80%;
        // margin: 0 auto;
        // padding: 10px;
    }

    .player {
      width: 400px;
      padding: 10px;
      // .borderRadius();
      background: @colorBlue;

      &__header {
        display: flex;
        align-items: center;
      }

      &__logo {
        width: 50px;
        height: 50px;
        margin-right: 10px;
      }

      &__heading {
        color: @colorYellow;
        font-family: 'Arial Rounded MT Bold', Arial, sans-serif;
      }

      &__file {
        display: block;
        width: 100%;
        height: 100%;
      }

      &__list {
        min-height: 200px;
        margin: 0 0 20px;
        padding: 0;
        &__i {
          width: 100%;
          height: 28px;
          padding: 5px 20px 5px 5px;
          margin-bottom: 5px;
          list-style: none;
          overflow: hidden;
          white-space: nowrap;
          text-overflow: ellipsis;
          color: @colorWhite;
          background: @colorLightBlue;
          cursor: pointer;
          box-sizing: border-box;
          .borderRadius();
          transition: background .3s ease-in;

          &.current,
          &:hover {
            background: @colorGrey;
          }

          &.dragging,
          &.over {
            position: relative;
            top: -2px;
            left: 5px;
            opacity: .5;
          }

          &.playing {
            position: relative;
            &:after {
              position: absolute;
              right: 0;
              top: 0;
              padding-right: 10px;
              width: 20%;
              height: 100%;
              text-align: right;
              line-height: 2;
              content: '\25B6';
            }
          }

          &.paused {
            position: relative;
            animation: blink 2s infinite;
            &:after {
              position: absolute;
              right: 0;
              top: 0;
              padding-right: 10px;
              width: 20%;
              height: 100%;
              text-align: right;
              line-height: 2;
              content: '\00275A\00275A';
            }
          }
        }
      }

      &__progressbar {
        width: 100%;
        height: 20px;
        display: flex;
        margin-bottom: 20px;
        .borderRadius();
        background: @colorLightBlue;
        cursor: pointer;
        overflow: hidden;

        &__scale {
          width: 90%;

          &__current {
            width: 0;
            height: 100%;
            background: @colorWhite;
          }
        }

        &__time {
          width: 10%;
          height: 100%;
          color: @colorWhite;
          padding: 3px 7px;
          box-sizing: border-box;
          font-size: 12px;
          background: @colorGrey
        }
      }

      &__ctrls {
        display: flex;
        justify-content: space-between;
      }

      &__ctrl {
        width: 18%;
        height: 30px;
        display: block;
        padding: 10px;
        color: @colorGrey;
        font-size: 20px;
        line-height: 30px;
        border: none;
        .borderRadius();
        cursor: pointer;
        background: @colorYellow;
        box-sizing: content-box;
        transition: background .3s ease-in;

        &:last-of-type {
          margin-right: 0;
        }

        &:hover {
          background: @colorLightBlue;
        }
      }
    }
</style>
