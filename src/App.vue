<template>
  <div id="app">
    <div id="mainWrapper">
      <div id="audioSelectWrapper">
        <div id="localFileBut">
          <span>Load local files</span>
        </div>
        <div id="micSelect" v-show="false">
          <span>Use Mic</span>
        </div>
      </div>
      <div id="presetControls">
        <div>Preset: <select id="presetSelect"></select></div>
        <div>Cycle: <input type="checkbox" id="presetCycle" checked="true"/>
                    <input type="number" id="presetCycleLength" step="1" value="15" min="1" /></div>
        <div>Random: <input type="checkbox" id="presetRandom" checked="true" /></div>
      </div>
      <canvas id='canvas' width='800' height='600'>
      </canvas>
    </div>
  </div>
</template>

<script>
/* global CCapture */
import HelloWorld from './components/HelloWorld.vue'
import _ from 'lodash'
import $ from 'jquery'
import butterchurn from 'butterchurn'
import butterchurnPresets from 'butterchurn-presets'


export default {
  name: 'app',
  components: {
    HelloWorld
  },
  mounted () {
    var visualizer = null;
    var rendering = false;
    var audioContext = null;
    var sourceNode = null;
    var delayedAudible = null;
    var cycleInterval = null;
    var presets = {};
    var presetKeys = [];
    var presetIndexHist = [];
    var presetIndex = 0;
    var presetCycle = true;
    var presetCycleLength = 15000;
    var presetRandom = true;
    var canvas = document.getElementById('canvas');
    var capturer = new CCapture( {
        format: 'ffmpegserver',
        framerate: 60,
        verbose: true,
        name: "foobar",     // videos will be named foobar-#.mp4, untitled if not set.
        extension: ".mp4",  // extension for file. default = ".mp4"
        codec: "mpeg4",     // this is an valid ffmpeg codec "mpeg4", "libx264", "flv1", etc...
                            // if not set ffmpeg guesses based on extension.
    });
    var numFrames = 500

    function connectToAudioAnalyzer(sourceNode) {
      if(delayedAudible) {
        delayedAudible.disconnect();
      }

      delayedAudible = audioContext.createDelay();
      delayedAudible.delayTime.value = 0.26;

      sourceNode.connect(delayedAudible)
      delayedAudible.connect(audioContext.destination);

      visualizer.connectAudio(delayedAudible);
    }

    var frameNum = 0
    function startRenderer() {
      visualizer.render();
      capturer.capture(canvas);
      frameNum++
      if (frameNum === numFrames) {
        capturer.stop()
        capturer.save(showVideoLink)
      } else {
        requestAnimationFrame(() => startRenderer());
      }
    }

    function showVideoLink(url, size) {
      size = size ? (" [size: " + (size / 1024 / 1024).toFixed(1) + "meg]") : " [unknown size]";
      var a = document.createElement("a");
      a.href = url;
      var filename = url;
      var slashNdx = filename.lastIndexOf("/");
      if (slashNdx >= 0) {
        filename = filename.substr(slashNdx + 1);
      }
      a.download = filename;
      a.appendChild(document.createTextNode(url + size));
      document.body.appendChild(a);
    }

    function playBufferSource(buffer) {
      if (!rendering) {
        rendering = true;
        startRenderer();
      }

      if (sourceNode) {
        sourceNode.disconnect();
      }

      sourceNode = audioContext.createBufferSource();
      sourceNode.buffer = buffer;
      connectToAudioAnalyzer(sourceNode);

      sourceNode.start(0);
    }

    function loadLocalFiles(files, index = 0) {
      audioContext.resume();

      var reader = new FileReader();
      reader.onload = (event) => {
        audioContext.decodeAudioData(
          event.target.result,
          (buf) => {
            playBufferSource(buf);

            setTimeout(() => {
              if (files.length > index + 1) {
                loadLocalFiles(files, index + 1);
              } else {
                sourceNode.disconnect();
                sourceNode = null;
                $("#audioSelectWrapper").css('display', 'block');
              }
            }, buf.duration * 1000);
          }
        );
      };

      var file = files[index];
      reader.readAsArrayBuffer(file);
    }

    function connectMicAudio(sourceNode, audioContext) {
      audioContext.resume();

      var gainNode = audioContext.createGain();
      gainNode.gain.value = 1.25;
      sourceNode.connect(gainNode);

      visualizer.connectAudio(gainNode);
      startRenderer();
    }

    function nextPreset(blendTime = 5.7) {
      presetIndexHist.push(presetIndex);

      var numPresets = presetKeys.length;
      if (presetRandom) {
        presetIndex = Math.floor(Math.random() * presetKeys.length);
      } else {
        presetIndex = (presetIndex + 1) % numPresets;
      }

      visualizer.loadPreset(presets[presetKeys[presetIndex]], blendTime);
      $('#presetSelect').val(presetIndex);
    }

    function prevPreset(blendTime = 5.7) {
      var numPresets = presetKeys.length;
      if (presetIndexHist.length > 0) {
        presetIndex = presetIndexHist.pop();
      } else {
        presetIndex = ((presetIndex - 1) + numPresets) % numPresets;
      }

      visualizer.loadPreset(presets[presetKeys[presetIndex]], blendTime);
      $('#presetSelect').val(presetIndex);
    }

    function restartCycleInterval() {
      if (cycleInterval) {
        clearInterval(cycleInterval);
        cycleInterval = null;
      }

      if (presetCycle) {
        cycleInterval = setInterval(() => nextPreset(2.7), presetCycleLength);
      }
    }

    $(document).keydown((e) => {
      if (e.which === 32 || e.which === 39) {
        nextPreset();
      } else if (e.which === 8 || e.which === 37) {
        prevPreset();
      } else if (e.which === 72) {
        nextPreset(0);
      }
    });

    $('#presetSelect').change(() => {
      presetIndexHist.push(presetIndex);
      presetIndex = parseInt($('#presetSelect').val());
      visualizer.loadPreset(presets[presetKeys[presetIndex]], 5.7);
    });

    $('#presetCycle').change(() => {
      presetCycle = $('#presetCycle').is(':checked');
      restartCycleInterval();
    });

    $('#presetCycleLength').change(() => {
      presetCycleLength = parseInt($('#presetCycleLength').val() * 1000);
      restartCycleInterval();
    });

    $('#presetRandom').change(() => {
      presetRandom = $('#presetRandom').is(':checked');
    });

    $("#localFileBut").click(function() {
      $("#audioSelectWrapper").css('display', 'none');

      var fileSelector = $('<input type="file" accept="audio/*" multiple />');

      fileSelector[0].onchange = function() {
        loadLocalFiles(fileSelector[0].files);
      }

      fileSelector.click();
    });

    $("#micSelect").click(() => {
      $("#audioSelectWrapper").css('display', 'none');

      navigator.getUserMedia({ audio: true }, (stream) => {
        var micSourceNode = audioContext.createMediaStreamSource(stream);
        connectMicAudio(micSourceNode, audioContext);
      }, () => {
        // console.log('Error getting audio stream from getUserMedia', err);
      });
    });

    function initPlayer() {

      capturer.start();


      audioContext = new AudioContext();

      const URL = 'http://127.0.0.1:8081/11%20Tragic%20Treasure.mp3'

      window.fetch(URL)
        .then(response => response.arrayBuffer())
        .then(arrayBuffer => audioContext.decodeAudioData(arrayBuffer,
          (buf) => {
            playBufferSource(buf);
          })
        )

      presets = {};
      Object.assign(presets, butterchurnPresets.getPresets());
      // Object.assign(presets, butterchurnPresetsExtra.getPresets());
      presets = _(presets).toPairs().sortBy(([k]) => k.toLowerCase()).fromPairs().value();
      presetKeys = _.keys(presets);
      presetIndex = Math.floor(Math.random() * presetKeys.length);

      var presetSelect = document.getElementById('presetSelect');
      for(var i = 0; i < presetKeys.length; i++) {
          var opt = document.createElement('option');
          opt.innerHTML = presetKeys[i].substring(0,60) + (presetKeys[i].length > 60 ? '...' : '');
          opt.value = i;
          presetSelect.appendChild(opt);
      }

      visualizer = butterchurn.createVisualizer(audioContext, canvas , {
        width: 800,
        height: 600,
        pixelRatio: window.devicePixelRatio || 1,
        textureRatio: 1,
      });
      nextPreset(0);
      cycleInterval = setInterval(() => nextPreset(2.7), presetCycleLength);
    }

    initPlayer();
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
