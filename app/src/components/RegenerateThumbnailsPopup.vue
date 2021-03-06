<template>
    <div
        class="overlay"
        v-if="isVisible">
        <div class="popup">
            <icon
                name="blank-image"
                size="xl"
                primaryColor="color-8" />

            <h1>
                Your theme or thumbnail settings has been changed.
            </h1>

            <p class="popup-info">
                Pressing the Regenerate thumbnails button will start generating new image sizes defined by your new theme.
            </p>

            <progress-bar
                :color="progressColor"
                :progress="progress"
                :stopped="progressIsStopped"
                :message="message" />

            <div class="buttons">
                <p-button
                    v-if="!regenerateIsDone"
                    @click.native="regenerate"
                    :disabled="regeneratingThumbnails"
                    type="medium no-border-radius half-width">
                    Regenerate thumbnails
                </p-button>

                <p-button
                    v-if="!regenerateIsDone"
                    @click.native="skip"
                    :disabled="regeneratingThumbnails"
                    type="medium no-border-radius half-width cancel-popup">
                    Skip regeneration
                </p-button>

                <p-button
                    v-if="regenerateIsDone"
                    @click.native="skip"
                    :disabled="regeneratingThumbnails"
                    type="medium no-border-radius full-width">
                    OK
                </p-button>
            </div>
        </div>
    </div>
</template>

<script>
import { ipcRenderer } from 'electron';

export default {
    name: 'regenerate-thumbnails-popup',
    data () {
        return {
            isVisible: false,
            message: '',
            progress: 0,
            progressColor: 'blue',
            progressIsStopped: false,
            regeneratingThumbnails: false,
            regenerateIsDone: false,
            savedSettingsCallbac: false
        };
    },
    mounted () {
        this.$bus.$on('regenerate-thumbnails-display', (config) => {
            this.isVisible = true;
            this.message = '';
            this.progress = 0;
            this.progressColor = 'blue';
            this.progressIsStopped = false;
            this.regeneratingThumbnails = false;
            this.regenerateIsDone = false;
            this.savedSettingsCallback = config.savedSettingsCallback || false;
        });

        document.body.addEventListener('keydown', this.onDocumentKeyDown);
    },
    methods: {
        skip () {
            this.isVisible = false;
            this.message = '';
            this.progress = 0;
            this.progressColor = 'blue';
            this.progressIsStopped = false;
            this.regeneratingThumbnails = false;
            this.regenerateIsDone = false;

            if (this.savedSettingsCallback) {
                this.$bus.$emit('regenerate-thumbnails-close', this.savedSettingsCallback);
            }
        },
        regenerate () {
            if (this.regeneratingThumbnails) {
                return;
            }

            this.regeneratingThumbnails = true;
            this.progressIsStopped = false;
            this.message = 'Regenerating thumbnails...';

            setTimeout(() => {
                ipcRenderer.send('app-site-regenerate-thumbnails', {
                    name: this.$store.state.currentSite.config.name
                });

                ipcRenderer.once('app-site-regenerate-thumbnails-error', (event, data) => {
                    this.progressColor = 'red';
                    this.progressIsStopped = true;
                    this.message = data.message;
                    this.regeneratingThumbnails = false;
                    this.regenerateIsDone = true;
                });

                ipcRenderer.on('app-site-regenerate-thumbnails-progress', (event, data) => {
                    this.progress = data.value;
                    this.message = 'Progress: ' + data.value + '%';
                });

                ipcRenderer.once('app-site-regenerate-thumbnails-success', (event, data) => {
                    this.progress = 100;
                    this.progressColor = 'green';
                    this.progressIsStopped = true;
                    this.message = 'All thumbnails have been created.';
                    this.regeneratingThumbnails = false;
                    this.regenerateIsDone = true;

                    if (this.savedSettingsCallback) {
                        this.skip();
                    }
                });
            }, 350);
        },
        onDocumentKeyDown (e) {
            if (e.code === 'Enter' && this.isVisible && !this.regeneratingThumbnails) {
                this.onEnterKey();
            }
        },
        onEnterKey () {
            if (this.regenerateIsDone) {
                this.skip();
            } else {
                this.regenerate();
            }
        }
    },
    beforeDestroy: function() {
        this.$bus.$off('regenerate-thumbnails-display');
        ipcRenderer.removeAllListeners('app-site-regenerate-thumbnails-error');
        ipcRenderer.removeAllListeners('app-site-regenerate-thumbnails-progress');
        ipcRenderer.removeAllListeners('app-site-regenerate-thumbnails-success');
        document.body.removeEventListener('keydown', this.onDocumentKeyDown);
    }
}
</script>

<style lang="scss" scoped>
@import '../scss/variables.scss';
@import '../scss/popup-common.scss';

.overlay {
    z-index: 100006;
}

.popup {  
    padding: 4rem 4rem 6rem 4rem;   
    width: 60rem;

    &-info {
        margin: -1.5rem 0 4rem;
    }
}

.message {
    color: var(--text-primary-color);
    font-weight: 400;
    margin: 0;
    padding: 4rem;
    position: relative;
    text-align: left;

    &.text-centered {
        text-align: center;
    }
}
.buttons {
    display: flex;
    margin: 0 -4rem -6rem -4rem;
    position: relative;
    text-align: center;
    top: 1px;
}
</style>
