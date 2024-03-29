<script lang="ts">
import { Component, Prop, Vue, Inject } from 'nuxt-property-decorator';
import { map, delay, tap, filter } from 'rxjs/operators';

import { ResourceProvider } from '~~/providers/resource.provider';
import { clamp, mapRange } from '~~/utils/math.util';
import { CursorService } from '~~/services/cursor.service';
import { ViewportProvider } from '~~/providers/viewport.provider';
import { PreloaderService } from '~~/services/preloader.service';
import { Browser } from '~~/utils/browser.util';

@Component<Preloader>({
  subscriptions(this) {
    return {
      loadingProgress: this.resources.progress$.pipe(
        tap((progress) => {
          this.currentProgress = progress;

          setTimeout(() => {
            this.updateTotalProgresss(progress, progress > 0.8 ? 2000 : 4000);
          }, 100);
        }),
      ),
    };
  },
})
export default class Preloader extends Vue {
  private preloader = PreloaderService.getInstance();
  private cursor = CursorService.getInstance();
  private viewport = ViewportProvider.getInstance();
  private resources = ResourceProvider.getInstance();
  private currentProgress: number = 0;
  private circleDown: boolean = false;
  private circleDownTimeout: any;
  private loadingProgress: number = 0;
  private totalProgress: number = 0;
  private clicked: boolean = false;
  private progressTween?: smoovy.Tween;
  private ready: boolean = false;
  private loaded: boolean = false;
  private touchDevice: boolean = false;
  private moveProg = { x: 0, y: 0 };
  private pointTween?: smoovy.Tween;
  private circleTween?: smoovy.Tween;
  private chromeNotice: boolean = false;

  @Inject('glConfig')
  private glConfig!: { enabled: boolean };

  public mounted() {
    if (process.browser) {
      this.touchDevice = smoovy.BrowserSupport.IS_TOUCH_DEVICE;
      this.chromeNotice = smoovy.BrowserSupport.IS_FIREFOX || Browser.IS_EDGE;
    }

    if (this.glConfig.enabled) {
      this.cursor.ready.then(() => {
        setTimeout(() => this.ready = true);

        // Skip preloading
        // this.handleLoadingDone();
        // document.documentElement.classList.add('preloader-ready');

        this.cursor.setPosition(
          this.viewport.size.width / 2,
          this.viewport.size.height / 2,
          true,
        );

        this.cursor.hide();

        document.documentElement.classList.add('preloader-mounted');
      });
    } else {
      setTimeout(() => this.ready = true);

      document.documentElement.classList.add('preloader-mounted');
    }

    this.viewport.changed(200)
      .pipe(
        filter(() => ! this.clicked),
      )
      .subscribe((size) => {
        this.cursor.setPosition(size.width / 2, size.height / 2, false);
      });

    this.resources.load();
  }

  private updateTotalProgresss(
    progress: number,
    duration: number = 4000,
  ) {
    if (this.progressTween) {
      this.progressTween.stop();
    }

    if (process.browser) {
      this.progressTween = smoovy.Tween.to(
        {
          value: this.totalProgress,
        },
        {
          value: progress,
        },
        duration,
        {
          easing: smoovy.easings.Quad.out,
          update: ({ value }) => {
            this.totalProgress = clamp(value, 0, 1);
            this.loaded = this.totalProgress === 1;

            if (this.loaded) {
              document.documentElement.classList.add('preloader-ready');

              if (smoovy.BrowserSupport.IS_MOBILE_OR_TABLET) {
                setTimeout(() => this.handleCircleDown());
              }
            }
          },
        },
      );
    }
  }

  private handleCircleDown() {
    if (this.loaded) {
      clearTimeout(this.circleDownTimeout);

      this.circleDown = true;
      this.circleDownTimeout = setTimeout(() => {
        if (this.loaded) {
          this.handleLoadingDone();
        }
      }, 500);
    }
  }

  private handleLoadingDone() {
    this.clicked = true;

    document.documentElement.classList.add('hide-cursor');

    setTimeout(() => {
      this.preloader.resolveLoaded();
      document.documentElement.classList.add('preloader-loaded');
    }, 100);

    setTimeout(() => {
      this.cursor.show();
      this.cursor.setPosition(
        this.viewport.size.width / 2,
        this.viewport.size.height / 2,
        false,
      );
    }, 498);
  }

  private handleCircleUp() {
    if ( ! this.clicked) {
      clearTimeout(this.circleDownTimeout);

      this.circleDown = false;
    }
  }

  private handleMouseMove(event: MouseEvent) {
    if ( ! this.clicked && smoovy && ! smoovy.BrowserSupport.IS_TOUCH_DEVICE) {
      const point = this.$refs.point as HTMLElement;
      const circle = this.$refs.circle as HTMLElement;

      const x = mapRange(
        event.clientX - this.viewport.size.width / 2,
        -this.viewport.size.width / 2,
        this.viewport.size.width / 2,
        -1,
        1
      ) * 90;

      const y = mapRange(
        event.clientY - this.viewport.size.height / 2,
        -this.viewport.size.height / 2,
        this.viewport.size.height / 2,
        -1,
        1
      ) * 90;

      if (this.pointTween) {
        this.pointTween.stop();
      }

      smoovy.Tween.to(
        this.moveProg,
        {
          x, y
        },
        500,
        {
          easing: smoovy.easings.Back.out,
          update: ({ x, y }) => {
            point.style.transform = `
              translate3d(${x}px, ${y}px, 0)
            `;

            circle.style.transform = `
              translate3d(${x * .08}px, ${y * .08}px, 0)
            `;
          }
        }
      );
    }
  }
}
</script>

<template>
  <div
    :class="{ loaded, clicked }"
    class="preloader-wrapper fx-layout fx-vertical fx-center-center"
    @mousemove="handleMouseMove"
  >
    <div class="chrome-notice" v-if="chromeNotice">
      <span>Use Google Chrome or Safari for the best experience</span>
    </div>
    <div
      class="circle-wrapper"
      :class="{ active: circleDown && loaded }"
      @mousedown="handleCircleDown"
      @mouseup="handleCircleUp"
      @mousecancel="handleCircleUp"
      @touchstart="handleCircleDown"
      @touchend="handleCircleUp"
    >
      <div class="circle-image-wrapper" ref="circle">
        <svg
          version="1.1"
          xmlns="http://www.w3.org/2000/svg"
          xmlns:xlink="http://www.w3.org/1999/xlink"
          preserveAspectRatio="xMidYMid meet"
          viewBox="0 0 80 80"
          width="70"
          height="70"
        >
          <defs>
            <path
              d="M80 40C80 62.08 62.08 80 40 80C17.92 80 0 62.08 0 40C0 17.92 17.92 0 40 0C62.08 0 80 17.92 80 40Z"
              id="j1tAiwRU5Y"
            ></path>
            <clipPath id="clipaRMILI0sk">
              <use xlink:href="#j1tAiwRU5Y" opacity="1"></use>
            </clipPath>
          </defs>
          <g>
            <g>
              <g>
                <g clip-path="url(#clipaRMILI0sk)">
                  <use
                    xlink:href="#j1tAiwRU5Y"
                    opacity="1"
                    fill-opacity="0"
                    :stroke-dasharray="`
                      ${ totalProgress * (2 * Math.PI * 40) },
                      ${2 * Math.PI * 40}
                    `"
                    stroke="#ff0d00"
                    stroke-width="4"
                    stroke-opacity="1"
                  ></use>
                </g>
              </g>
            </g>
          </g>
        </svg>
      </div>
    </div>
    <div class="circle-inner-wrapper" :class="{ active: circleDown && loaded }">
      <div class="pulse"></div>
      <div class="point" :class="{ show: totalProgress > 0.5 }">
        <div class="point-inner" ref="point"></div>
      </div>
    </div>
    <span
      class="teaser-label"
      :class="{ active: ready && currentProgress >= 1 }"
    >
      <span>{{touchDevice ? 'Touch' : 'Click'}} & Hold</span>
    </span>
  </div>
</template>

<style scoped lang="scss">
.chrome-notice {
  position: absolute;
  left: 50%;
  bottom: 5%;
  transform: translate3d(-50%, 0, 0);
  text-align: center;

  @include fluid-size(font-size, 14px, 20px);
}

.preloader-wrapper {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 200;
  transition: bottom 1s;
  overflow: hidden;
  background-color: transparent;

  &.clicked {
    visibility: hidden;
    transition: visibility 0s .5s;

    .is-mobile & {
      opacity: 0;
      transition: visibility 0s .5s, opacity .5s;
    }
  }

  .circle-wrapper {
    position: relative;
    z-index: 2;
    user-select: none;
    opacity: 1;
    transition: transform .5s $ease-in-out-circ;
    transform: scale(.8) translateZ(0);

    &.active {
      transform: scale(1) translateZ(0);
    }
  }

  .circle-wrapper:hover .circle-image-wrapper,
  .circle-wrapper:hover + .circle-inner-wrapper .point-inner,
  .circle-wrapper.active .circle-image-wrapper,
  .circle-wrapper.active + .circle-inner-wrapper .point-inner {
    transform: none !important;
    transition: transform .3s;
  }

  @keyframes rotate360 {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }

  &:not(.loaded) .circle-wrapper svg {
    animation: rotate360 3s forwards linear infinite;
  }

  .teaser-label {
    position: absolute;
    top: calc(50% + 50px);
    left: 50%;
    transform: translate(-50%, -50%);
    text-transform: uppercase;
    letter-spacing: 1px;
    color: $color-black;
    padding: 2px 0;
    overflow: hidden;
    font-family: $font-neue-haas-light;

    html.is-mobile & {
      display: none;
    }

    @include fluid-size(font-size, 12px, 14px);

    span {
      display: block;
      transform: translate3d(0, 100%, 0);
      opacity: 0;
      transition:
        transform .5s $ease-out-smooth,
        opacity .5s;
    }
  }

  &:not(.clicked) .teaser-label.active span {
    opacity: 1;
    transform: translate3d(0, 0, 0);
    transition:
        transform 1.2s 1.5s $ease-out-smooth,
        opacity .5s 1.5s;
  }

  .point {
    position: absolute;
    left: 50%;
    top: 50%;
    width: 100%;
    height: 100%;
    will-change: transform;
    transform: translate3d(-50%, -50%, 0) scale(0);
    transition: transform 1s $ease-in-out-circ;

    .point-inner {
      width: 100%;
      height: 100%;
      border-radius: 50%;
      background-color: $color-red;

      html.is-mobile & {
        display: none;
      }
    }

    &.show {
      transform: translate3d(-50%, -50%, 0) scale(0.11);
    }
  }

  .circle-inner-wrapper {
    position: absolute;
    z-index: 0;
    left: 50%;
    top: 50%;
    width: 70px;
    height: 70px;
    transform: translate3d(-50%, -50%, 0);
  }

  .pulse {
    position: absolute;
    left: 50%;
    top: 50%;
    width: 100%;
    height: 100%;
    background-color: transparent;
    border: 35px solid $color-red;
    opacity: 0;
    transform: translate3d(-50%, -50%, 0) scale(0.11);
    border-radius: 50%;
    transition:
      transform 1s $ease-in-out-circ,
      border-width 1s $ease-in-out-circ,
      opacity .3s $ease-out-smooth;
  }

  .circle-inner-wrapper.active .pulse {
    border-width: 2px;
    opacity: 1;
    transform: translate3d(-50%, -50%, 0) scale(1);
    transition:
      transform 1s $ease-in-out-circ,
      border-width 1s $ease-in-out-circ,
      border-width .8s .2s $ease-in-out-circ;
  }
}
</style>
