<template>
  <div class="story">
    <div ref="timeline" class="timeline">
      <div class="slice" v-for="i in slides" :key="i">
        <div ref="progress" class="progress">&nbsp;</div>
      </div>
    </div>
    <div class="slide" :style="{background: slideStyle}">
      <img ref="screenSide" :src="require('../assets/' + slides[currentSlideIndex] + '.jpg')" alt="">
    </div>
  </div>
</template>
<script>
  import anime from 'animejs/lib/anime.es.js';
  import Hammer from 'hammerjs';
  import { EventBus } from '../helpers/EventBus.js';
  import FastAverageColor from 'fast-average-color';
  const SLIDE_DURATION = 2000;
  export default {
    name: 'Story',
    props: {
      slides: Array
    },
    data() {
      const timeline = anime.timeline({
        autoplay: false,
        duration: SLIDE_DURATION,
        easing: 'linear'
      });
      return {
        slideStyle: '',
        currentSlideIndex: 0,
        isActive: false,
        timeline: timeline,
        muted: true
      }
    },
    methods: {
      activate: function() { // Start timer
        this.resetSlide();
      },
      deactivate: function() {
        this.timeline.pause();
      },
      resetSlide: function() { // Jump to beginning of the slide
        this.timeline.pause();
        this.timeline.seek(this.currentSlideIndex * SLIDE_DURATION);
        this.timeline.play();
      },
      nextSlide: function() {
        if (this.currentSlideIndex < this.slides.length - 1) {
          this.currentSlideIndex++;
          this.changeColor()
          this.resetSlide();
        } else {
          this.nextStory();
        }
      },
      previousSlide: function() {
        if (this.currentSlideIndex > 0) {
          this.currentSlideIndex--;
          this.resetSlide();
        } else {
          this.previousStory();
        }
      },
      nextStory: function() {
        EventBus.$emit('NEXT_STORY');
      },
      previousStory: function() {
        EventBus.$emit('PREVIOUS_STORY');
      },
      changeColor() {
        const fac = new FastAverageColor();
        let container = this.$refs.screenSide;
        fac.getColorAsync(container, {algorithm: 'dominant'})
                .then(color => {
                  this.slideStyle = color.hex
                })
                .catch(e => {
                  console.error(e);
                });
      }
    },
    mounted() {
      // Add progress bars to the timeline animation group
      this.slides.forEach((color, index) => {
        this.timeline.add({
          targets: this.$refs.progress[index],
          width: '100%',
          changeBegin: () => {
            // Update the Vue componenet state when progress bar begins to play
            this.currentSlideIndex = index;
            this.changeColor()
          },
          complete: () => {
            // Move to the next story when finished playing all slides
            if (index === this.slides.length - 1) {
              this.nextStory();
            }
          },
        });
      });
      this.hammer = new Hammer.Manager(this.$el, {
        recognizers: [
          [Hammer.Pan, { direction: Hammer.DIRECTION_HORIZONTAL }],
          [Hammer.Tap],
          [Hammer.Press, { time: 1, threshold: 1000000 }]
        ]
      })
      this.hammer.on("press", () => {
        this.timeline.pause();
      });
      this.hammer.on("pressup tap", () => {
        this.timeline.play();
      });
      // Tap on the side to navigate between slides
      this.hammer.on("tap", event => {
        this.muted = !this.muted
        if (event.center.x > window.innerWidth / 3) {
          this.nextSlide();
        } else {
          this.previousSlide();
        }
      });
      // Handle swipe
      this.hammer.on("pan", event => {
        if (event.isFinal) {
          if (event.deltaX < 0) {
            this.nextStory();
          } else if (event.deltaX > 0) {
            this.previousStory();
          }
        }
      });
    }
  }
</script>
<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
  .story {
    float: left;
    position: relative;
    height: 100vh;
    width: 100vw;
    z-index: 1;
    display: flex;
    flex-direction: column;
  }
  .timeline {
    position: absolute;
    display: flex;
    flex-grow: 0;
    width: 100%;
    border-radius: 5px;
  }
  .timeline > .slice {
    background: rgba(255, 255,
    255, 0.4);;
    height: 3px;
    margin: 10px 3px;
    width: 100%;
    border-radius: 5px;
  }
  .timeline > .slice > .progress {
    background: #fff;
    height: 3px;
    width: 0%;
    border-radius: 5px;
  }
  .slide {
    display: flex;
    align-items: center;
    justify-content: center;
  }
  .slide img {
    width: 100vw;
    height: 100vh;
    object-fit: contain;
  }
  .slide video {
    width: 100vw;
    height: 100vh;
    object-fit: contain;
  }
</style>