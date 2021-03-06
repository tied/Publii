<template>
    <div class="overlay" v-if="isVisible">
        <div class="popup">
            <h1>Rendering ...</h1>
            <p class="popup-info">
                Please wait while the rendering process is completed.
            </p>

            <progress-bar
                v-if="!isPostPreview && !isHomepagePreview"
                :color="progressColor"
                :progress="progress"
                :stopped="progressIsStopped"
                :message="messageFromRenderer" />
        </div>
    </div>
</template>

<script>
import { ipcRenderer } from 'electron';
import Utils from './../helpers/utils.js';

export default {
    name: 'rendering-popup',
    data: function() {
        return {
            isVisible: false,
            isPostPreview: false,
            isHomepagePreview: false,
            isTagPreview: false,
            isAuthorPreview: false,
            messageFromRenderer: '',
            progress: 0,
            progressColor: 'blue',
            progressIsStopped: false
        };
    },
    mounted: function() {
        this.$bus.$on('rendering-popup-display', (config) => {
            this.isVisible = true;
            this.messageFromRenderer = '';
            this.progress = 0;
            this.progressColor = 'blue';
            this.progressIsStopped = false;
            this.isPostPreview = false;
            this.isHomepagePreview = false;
            this.isTagPreview = false;
            this.isAuthorPreview = false;

            if (config && config.homepageOnly) {
                this.isHomepagePreview = true;
                this.runRenderingPreview(false, 'home');
            } else if (config && config.tagOnly) {
                this.isTagPreview = true;
                this.runRenderingPreview(config, 'tag');
            } else if (config && config.authorOnly) {
                this.isAuthorPreview = true;
                this.runRenderingPreview(config, 'author');
            } else if (config && config.postOnly) {
                this.isPostPreview = true;
                this.runRenderingPreview(config, 'post');
            } else {
                this.runRenderingPreview();
            }
        });

        ipcRenderer.on('app-rendering-progress', this.renderingProgress);
    },
    methods: {
        runRenderingPreview (itemConfig = false, mode = false) {
            if(!this.themeIsSelected) {
                this.$bus.$emit('confirm-display', {
                    message: 'You have to select a theme before trying to create a preview of your website. Please go to the website settings and select a theme.',
                    okLabel: 'Go to settings',
                    okClick: () => {
                        let siteName = this.$route.params.name;
                        this.$route.push('/site/' + siteName + '/settings/');
                    }
                });

                return;
            }

            let renderConfig = {
                "site": this.$store.state.currentSite.config.name,
                "theme": this.$store.state.currentSite.config.theme,
                "ampIsEnabled": this.$store.state.currentSite.config.advanced.ampIsEnabled
            };

            if (mode === 'post' && itemConfig) {
                renderConfig.mode = 'post';
                renderConfig.itemID = itemConfig.itemID;
                renderConfig.postData = itemConfig.postData;
                renderConfig.source = 'post-editor';
            } else if (mode === 'home') {
                renderConfig.mode = 'home';
            } else if (mode === 'tag') {
                renderConfig.mode = 'tag';
                renderConfig.itemID = itemConfig.itemID;
            } else if (mode === 'author') {
                renderConfig.mode = 'author';
                renderConfig.itemID = itemConfig.itemID;
            }

            ipcRenderer.send('app-preview-render', renderConfig);

            ipcRenderer.once('app-preview-rendered', (event, data) => {
                if(data.status === true) {
                    ipcRenderer.send('app-preview-show', {
                        "site": this.$store.state.currentSite.config.name,
                        "ampIsEnabled": this.$store.state.currentSite.config.advanced.ampIsEnabled
                    });

                    if (mode === 'post' || mode === 'home' || mode === 'tag' || mode === 'author') {
                        setTimeout(() => {
                            this.isVisible = false;
                        }, 500);
                    }
                } else {
                    this.$bus.$emit('alert-display', {
                        message: 'An error occured during creating the preview.'
                    });
                }
            });

            ipcRenderer.removeListener('app-preview-render-error', this.renderError);
            ipcRenderer.once('app-preview-render-error', this.renderError);
        },
        renderingProgress: function(event, data) {
            this.messageFromRenderer = data.message + ' - ' + data.progress + '%';
            this.progress = data.progress;

            if(this.progress === 100) {
                this.progressColor = 'green';
                this.progressIsStopped = true;
                this.messageFromRenderer = '';

                setTimeout(() => {
                    this.isVisible = false;
                }, 500);
            }
        },
        renderError(event, data) {
            let errorsHTML = Utils.generateErrorLog(data.message);
            let errorsText = Utils.generateErrorLog(data.message, true);

            this.$bus.$emit('error-popup-display', {
                errors: errorsHTML,
                text: errorsText
            });

            this.isVisible = false;
        },
        themeIsSelected() {
            return !(!this.$store.state.currentSite.config.theme || this.$store.state.currentSite.config.theme === '');
        }
    },
    beforeDestroy: function() {
        this.$bus.$off('rendering-popup-display');
        ipcRenderer.removeAllListeners('app-preview-render-error');
        ipcRenderer.removeAllListeners('app-rendering-progress');
    }
}
</script>

<style lang="scss" scoped>
@import '../scss/variables.scss';
@import '../scss/popup-common.scss';

.popup {  
    padding: 4rem 4rem 1rem 4rem;   
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
</style>
