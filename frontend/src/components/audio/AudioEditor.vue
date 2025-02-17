<template>
  <div class="column w-full no-wrap">
    <div class="action-tabs flex-center gap-4 shadow-1">
      <BaseButton
        v-for="action in actions"
        :key="action.key"
        :class="{ 'shadow-14': selectedAction === action.key }"
        :color="selectedAction === action.key ? 'primary' : 'blue-grey-7'"
        :tooltip="action.tooltip"
        :flat="selectedAction !== action.key"
        @click="selectAction(action.key)"
      >
        <q-icon :name="action.icon" size="18px" />
      </BaseButton>
      <BaseButton color="blue-grey-7" tooltip="Close" flat @click="showCloseAudioDialog">
        <q-icon name="close" size="18px" />
      </BaseButton>
    </div>

    <div class="waveform-wrapper">
      <div id="waveform" class="waveform" />

      <div class="waveform__time flex-center text-primary text-body1 text-weight-medium">
        {{ formatTime(region[0]) }}
        —
        {{ formatTime(region[1]) }}
      </div>
    </div>

    <div class="bottom-section column items-center gap-16 q-py-xl shadow-1">
      <BaseButton
        class="shadow-14"
        color="primary"
        size="20px"
        padding="sm xl"
        tabindex="0"
        unelevated
        @click="wavesurfer.playPause()"
      >
        <transition enter-active-class="animated zoomIn" leave-active-class="animated zoomOut" mode="out-in">
          <q-icon v-if="wavesurfer.isPlaying()" class="play-icon" name="pause" />
          <q-icon v-else class="play-icon" name="play_arrow" />
        </transition>
      </BaseButton>

      <div v-show="selectedAction === 'volume'">
        <AudioEditorSliderVolume :model-value="volume" label="Browser volume" @update:model-value="setVolume" />
        <AudioEditorSliderVolume v-model="exportedVolume" label="Export volume" />
      </div>
      <div v-show="selectedAction === 'speed'">
        <div class="text-subtitle2 text-center">
          Average BPM ~{{ musicInfo?.tempo ? Math.round(musicInfo?.tempo * (speed / 100)) : 'Undefined' }}
        </div>
        <AudioEditorSlider
          :model-value="speed"
          label="Speed"
          :label-value="speed / 100"
          :min="10"
          :max="300"
          :step="10"
          @update:model-value="setSpeed"
        />
      </div>
      <div v-show="selectedAction === 'bitrate'">
        <AudioEditorSlider v-model="bitrate" label="Export bitrate" :min="16" :max="320" />
      </div>
      <div v-show="selectedAction === 'equalizer'" class="row gap-4">
        <div class="row gap-2">
          <AudioEditorSlider
            v-for="(item, index) in wavesurferFilters"
            :key="item.frequency.value"
            :model-value="equalizer[index].value"
            :label="item.frequency.value"
            :min="-12"
            :max="12"
            vertical
            reverse
            @update:model-value="updateEqualizer($event, index)"
          />
        </div>
        <BaseButton
          class="self-center"
          style="margin-bottom: 28px"
          icon="restart_alt"
          tooltip="Reset EQ"
          @click="resetEqualizer"
        />
      </div>
      <BaseButton class="shadow-14" label="Export" color="primary" padding="sm xl" unelevated @click="exportAudio" />
    </div>

    <BaseDialog
      title="Finding song BPM"
      confirm-text="Confirm"
      confirm-classes="full-width"
      :confirm-loading="isTempoLoading"
      :model-value="dialog.openedName.value === 'findBPM'"
      hide-close-button
      hide-close-icon
    >
      Please wait 5-20 seconds
    </BaseDialog>
    <BaseDialog
      type="delete"
      title="Close current audio"
      confirm-text="Confirm"
      :model-value="dialog.openedName.value === 'closeConfirm'"
      @close="dialog.close"
      @confirm="$emit('close')"
    >
      Are you sure you want to close this file?
    </BaseDialog>
  </div>
</template>

<script lang="ts">
import { defineComponent, onBeforeUnmount, onMounted, ref } from 'vue';

import AudioEditorSliderVolume from 'components/audio/AudioEditorSliderVolume.vue';

import WaveSurfer from 'wavesurfer.js';
import Regions from 'wavesurfer.js/dist/plugin/wavesurfer.regions';
// import audioEncoder from 'audio-encoder';
// import { Mp3Encoder } from 'lamejs';
import MusicTempo from 'music-tempo';

import AudioEditorSlider from 'components/audio/AudioEditorSlider.vue';
import { WaveSurferBackend } from 'wavesurfer.js/types/backend';
import { EventTargetWithResult } from 'src/models/common/eventTargetWithResult.model';
import { MusicTempoData } from 'app/src';
import useDialog from 'src/composables/common/useDialog';

interface EqItem {
  f: number;
  type: BiquadFilterType;
  value: number;
}

export default defineComponent({
  name: 'AudioEditor',

  components: {
    AudioEditorSlider,
    AudioEditorSliderVolume,
  },

  props: {
    rawAudio: {
      type: File,
      default: null,
    },
    rawAudioDuration: {
      type: Number,
      default: null,
    },
  },

  emits: ['close'],

  setup(props) {
    const dialog = useDialog();

    const wavesurfer = ref<WaveSurfer | null>(null);
    function formatTime(v: number) {
      const minutes = Math.floor(v / 60);
      const formattedMinutes = Number(minutes) < 10 ? `0${minutes}` : minutes;

      const seconds = Math.floor(v % 60);
      const formattedSeconds = Number(seconds) < 10 ? `0${seconds}` : seconds;

      const ms = String(v.toFixed(1)).split('.')[1];
      return `${formattedMinutes}:${formattedSeconds}.${ms}`;
    }

    function decodeAndSetMusicInfo() {
      isTempoLoading.value = true;

      const context = new AudioContext();
      const reader = new FileReader();
      reader.onload = async ($event) => {
        const target = $event.target as EventTargetWithResult;
        const result = target.result as unknown as ArrayBuffer;
        await context.decodeAudioData(result, (buffer: AudioBuffer) => {
          let audioData: number[] | Float32Array = [];
          // average of the two channels
          if (buffer.numberOfChannels == 2) {
            let channel1Data = buffer.getChannelData(0);
            let channel2Data = buffer.getChannelData(1);
            let length = channel1Data.length;
            for (let i = 0; i < length; i++) {
              audioData[i] = (channel1Data[i] + channel2Data[i]) / 2;
            }
          } else {
            audioData = buffer.getChannelData(0);
          }

          try {
            // TODO
            // eslint-disable-next-line @typescript-eslint/no-unsafe-call
            musicInfo.value = new MusicTempo(audioData) as MusicTempoData;
          } finally {
            isTempoLoading.value = false;
            dialog.close();
          }
        });
      };
      reader.readAsArrayBuffer(props.rawAudio);
    }
    const isTempoLoading = ref(false);
    const musicInfo = ref<MusicTempoData | null>(null);

    const volume = ref(10);
    const exportedVolume = ref(100);
    function setVolume(v = 10) {
      volume.value = v;
      wavesurfer.value?.setVolume(volume.value / 100);
    }

    const speed = ref(100);
    function setSpeed(v: number) {
      speed.value = v;
      wavesurfer.value?.setPlaybackRate(speed.value / 100);
    }

    // TODO: set current audio bitrate and limit slider max?
    // didn't found how to get bitrate in js, only duration, size and samplerate
    const bitrate = ref(192);

    const equalizer = ref<EqItem[]>([
      {
        f: 32,
        type: 'lowshelf',
        value: 0,
      },
      {
        f: 64,
        type: 'peaking',
        value: 0,
      },
      {
        f: 125,
        type: 'peaking',
        value: 0,
      },
      {
        f: 250,
        type: 'peaking',
        value: 0,
      },
      {
        f: 500,
        type: 'peaking',
        value: 0,
      },
      {
        f: 1000,
        type: 'peaking',
        value: 0,
      },
      {
        f: 2000,
        type: 'peaking',
        value: 0,
      },
      {
        f: 4000,
        type: 'peaking',
        value: 0,
      },
      {
        f: 8000,
        type: 'peaking',
        value: 0,
      },
      {
        f: 16000,
        type: 'highshelf',
        value: 0,
      },
    ]);
    const wavesurferFilters = ref<BiquadFilterNode[] | null>(null);
    function updateEqualizer(v: number, index: number) {
      if (!wavesurferFilters.value) return;
      wavesurferFilters.value[index].gain.value = v;
      equalizer.value[index].value = v;
    }
    function resetEqualizer() {
      wavesurferFilters.value?.forEach((f) => (f.gain.value = 0));
      equalizer.value?.forEach((f) => (f.value = 0));
    }

    onMounted(() => {
      wavesurfer.value = WaveSurfer.create({
        container: '#waveform',
        waveColor: 'rgb(207,217,237)',
        progressColor: '#60A5FA',
        normalize: true,
        // barRadius: 50,
        // barGap: 50,
        // barHeight: 50,
        plugins: [
          // eslint-disable-next-line @typescript-eslint/no-unsafe-call,@typescript-eslint/no-unsafe-member-access
          Regions.create(),
        ],
      });
      wavesurfer.value?.loadBlob(props.rawAudio);
      setVolume();

      wavesurfer.value.on('ready', () => {
        // eslint-disable-next-line @typescript-eslint/no-unsafe-call
        wavesurfer.value?.addRegion({
          end: props.rawAudioDuration,
          loop: true,
        });
        wavesurferFilters.value = equalizer.value.map((band) => {
          const { ac } = wavesurfer.value?.backend as WaveSurferBackend & { ac: AudioContext };
          let filter = ac.createBiquadFilter();
          filter.type = band.type;
          filter.gain.value = 0;
          filter.Q.value = 1;
          filter.frequency.value = band.f;
          return filter;
        });

        wavesurfer.value?.backend.setFilters(wavesurferFilters.value);
      });

      wavesurfer.value?.on('region-update-end', updateExportRegion);

      // TODO: event don't handle sometimes cuz we have focus on button
      addEventListener('keydown', handleKeyPress);
    });
    onBeforeUnmount(() => {
      removeEventListener('keydown', handleKeyPress);
    });
    function handleKeyPress($event: KeyboardEvent) {
      const code = $event.code;
      if (code === 'Space') {
        void wavesurfer.value?.playPause();
        $event.preventDefault();
      } else if (code === 'ArrowLeft') {
        wavesurfer.value?.skipBackward(3);
        $event.preventDefault();
      } else if (code === 'ArrowRight') {
        wavesurfer.value?.skipForward(3);
        $event.preventDefault();
      }
    }

    const region = ref([0, props.rawAudioDuration]);
    function updateExportRegion(updatedRegion: { start: number; end: number }) {
      if (updatedRegion.start !== region.value[0]) wavesurfer.value?.play(updatedRegion.start, updatedRegion.end);
      region.value = [updatedRegion.start, updatedRegion.end];
    }

    async function exportAudio() {
      try {
        // TODO: fix err https://github.com/zhuker/lamejs/issues/86

        const audioContext = new AudioContext();
        const buffer = await new Response(props.rawAudio).arrayBuffer();

        const decodedData = await audioContext.decodeAudioData(buffer);
        console.log('decodedData', decodedData);

        // const length = 44100; // one second @ 44.1KHz
        // const audioBuffer = audioContext.createBuffer(1, length, 44100);
        // const channelData = audioBuffer.getChannelData(0);
        //
        // // fill some audio
        // for (let i = 0; i < length; i++) {
        //   channelData[i] = Math.sin(i * 0.03);
        // }
        //
        // // convert as mp3 and save file using file-saver
        // audioEncoder(audioBuffer, 128, null, (blob) => {
        //   const url = URL.createObjectURL(blob);
        //   console.log('MP3 URl: ', url);
        // });

        // const mp3Encoder = new Mp3Encoder(decodedData.numberOfChannels, decodedData.sampleRate, 128);
        // const mp3Data: string[] = [];
        //
        // console.log(mp3Encoder);
        //
        // const left = new Int16Array(44100); //one second of silence (get your data from the source you have)
        // const right = new Int16Array(44100); //one second of silence (get your data from the source you have)
        // const samples = new Int16Array(44100); //one second of silence (get your data from the source you have)
        // const sampleBlockSize = 1152; //can be anything but make it a multiple of 576 to make encoders life easier
        //
        // for (let i = 0; i < samples.length; i += sampleBlockSize) {
        //   const leftChunk = left.subarray(i, i + sampleBlockSize);
        //   const rightChunk = right.subarray(i, i + sampleBlockSize);
        //   const encoded = mp3Encoder.encodeBuffer(leftChunk, rightChunk);
        //   if (encoded.length > 0) {
        //     mp3Data.push(encoded);
        //   }
        // }
        // const mp3 = mp3Encoder.flush();
        // if (mp3.length > 0) {
        //   mp3Data.push(mp3);
        // }
        //
        // console.log('mp3Data', mp3Data);
        //
        // const blob = new Blob(mp3Data, { type: 'audio/mp3' });
        // console.log('blob', blob);
        // const url = URL.createObjectURL(blob);
        // console.log('MP3 URl: ', url);
      } catch (e) {
        console.error(e);
      }
    }

    const actions = [
      {
        tooltip: 'Volume',
        key: 'volume',
        icon: 'volume_up',
      },
      {
        tooltip: 'Speed',
        key: 'speed',
        icon: 'speed',
      },
      {
        tooltip: 'Bitrate',
        key: 'bitrate',
        icon: 'grain',
      },
      {
        tooltip: 'Equalizer',
        key: 'equalizer',
        icon: 'equalizer',
      },
    ];
    function selectAction(key: string) {
      selectedAction.value = key;
      if (key === 'speed') {
        wavesurfer.value?.pause();
        dialog.open('findBPM');
        decodeAndSetMusicInfo();
      }
    }
    const selectedAction = ref('volume');

    function showCloseAudioDialog() {
      wavesurfer.value?.pause();
      dialog.open('closeConfirm');
    }

    return {
      dialog,

      wavesurfer,
      formatTime,
      isTempoLoading,
      musicInfo,

      volume,
      exportedVolume,
      setVolume,

      speed,
      setSpeed,

      bitrate,

      equalizer,
      wavesurferFilters,
      updateEqualizer,
      resetEqualizer,

      region,
      exportAudio,

      actions,
      selectAction,
      selectedAction,

      showCloseAudioDialog,
    };
  },
});
</script>

<style lang="scss" scoped>
.waveform-wrapper {
  padding: 64px 32px 0 32px;
  background: #d4dded2e;

  .waveform__time {
    padding: 20px 0;
  }

  ::v-deep .waveform > wave {
    border-radius: 12px;
    > wave {
      border-right-color: #ff2a6f !important;
    }
  }

  ::v-deep .wavesurfer-region {
    background-color: rgba($primary, 0.2) !important;
    border-radius: 12px !important;
    box-shadow: lighten($primary, 20%) 0 0 32px 7px, #738caa7a 0 0 0 10000px;
  }
  ::v-deep .wavesurfer-handle {
    background-color: darken($primary, 5%) !important;
    width: 16px !important;
    cursor: grab !important;
    &:hover {
      background-color: darken($primary, 7%) !important;
    }
    &:active {
      cursor: grabbing !important;
      background-color: darken($primary, 10%) !important;
    }
  }
  ::v-deep .wavesurfer-handle-start {
    border-radius: 12px 0 0 12px !important;
  }
  ::v-deep .wavesurfer-handle-end {
    border-radius: 0 12px 12px 0 !important;
  }
}

.action-tabs {
  padding: 32px 16px;
}

.bottom-section {
  padding-left: 32px;
  padding-right: 32px;
  height: 100%;
}

.play-icon {
  --animate-duration: 50ms;
}
</style>
