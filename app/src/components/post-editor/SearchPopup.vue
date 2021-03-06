<template>
    <div :class="{ 'search-popup': true, 'is-visible': isVisible }">
        <input 
            @keyup="doKeyboardNavigation"
            ref="search-phrase-input"
            spellcheck="false"
            v-model="searchPhrase" />
        <span>
            <template v-if="resultsCount > 0">
                {{ currentResultIndex }} / 
            </template> 
            {{ resultsCount }}
        </span>        
        <button @click.prevent="getPrevResult()">
            <icon
                size="m"
                name="arrow-up"  />
        </button>
        <button @click.prevent="getNextResult()">
            <icon
                size="m"
                name="arrow-down"  />
        </button>
        <button @click.prevent="finishSearch()">
            <icon
                size="m"
                name="close"  />
        </button>
    </div>
</template>

<script>
import { ipcRenderer, remote } from 'electron';

export default {
    name: 'search-popup',
    data () {
        return {
            webviewContents: null,
            searchPhrase: '',
            isVisible: false,
            currentResultIndex: 0,
            resultsCount: 0
        };
    },
    watch: {
        searchPhrase (newValue) {
            if (this.isVisible && newValue.trim() !== '') {
                this.webviewContents.findInPage(newValue);
            } else {
                this.resultsCount = 0;
                this.currentResultIndex = 0;
                this.webviewContents.stopFindInPage('clearSelection');
            }
        }
    },
    mounted () {
        document.querySelector('webview').addEventListener('dom-ready', () => {
            this.webviewContents = remote.webContents.fromId(document.querySelector('webview').getWebContentsId());

            document.querySelector('webview').addEventListener('found-in-page', e => {
                this.currentResultIndex = e.result.activeMatchOrdinal;
                this.resultsCount = e.result.matches;
            });
        });

        ipcRenderer.on('app-show-search-form', this.showSearch);
        this.$bus.$on('app-show-search-form', this.showSearch);
    },
    methods: {
        getNextResult () {
            if (this.resultsCount === 0) {
                return;
            }

            this.webviewContents.findInPage(this.searchPhrase);
        },
        getPrevResult () {
            if (this.resultsCount === 0) {
                return;
            }

            this.webviewContents.findInPage(this.searchPhrase, {
                forward: false
            });
        },
        showSearch () {
            this.isVisible = true;
            this.$refs['search-phrase-input'].focus();
        },
        finishSearch () {
            this.webviewContents.stopFindInPage('clearSelection');
            this.isVisible = false;
        },
        doKeyboardNavigation (e) {
            if (e.code === 'Enter' && !event.isComposing) {
                this.getNextResult();
            }
        }
    },
    beforeDestroy () {
        ipcRenderer.removeAllListeners('app-show-search-form');
        this.$bus.$off('app-show-search-form', this.showSearch);
    }
};
</script>

<style lang="scss" scoped>
@import '../../scss/variables.scss';
@import '../../scss/mixins.scss';

.search-popup {
    background: var(--popup-bg);
    border-radius: 6px;
    box-shadow: 0 0 5px rgba(black, .125);
    left: 50%;
    opacity: 0;
    padding: 15px 22px 15px 30px;
    pointer-events: none;
    position: fixed;
    top: 20px;
    transform: translateX(-50%);
    transition: var(--transition);
    width: auto;
    z-index: 10000000;

    &.is-visible {
        opacity: 1;
        pointer-events: auto;
        top: 40px;
    }

    input {
        background: none;
        border: none;
        border-bottom: 1px solid var(--input-border-focus);        
        color: var(--text-primary-color);
        font-size: 18px;
        font-weight: 500;
        width: 300px;
    }
    
    span {
        color: var(--text-primary-color);
        display: inline-block;
        font-size: 12px;  
        margin-right: 10px;
        min-width: 3.5rem;
    }
    
    button {
        background: none;
        border: none;
        border-radius: 50%;        
        cursor: pointer;        
        height: 2.8rem;         
        padding: 0;
        position: relative;       
        text-align: center;       
        transition: var(--transition);             
        width: 2.8rem;
        
        & > svg {           
            stroke: var(--icon-secondary-color);
            vertical-align: middle;
        }       
        
        &:active,
        &:focus,
        &:hover {
            & > svg {           
                stroke: var(--icon-tertiary-color);
            }
        }
        
        &:hover {
            background: var(--input-border-color);            
        }  
    }
}
</style>
