<template>
    <div style="display:flex;flex-direction:column;height:100%;padding:1px" :class="'platform-' + platform" @auxclick.prevent>
        <div v-html="styling"></div>
        <div style="display:flex;align-items:stretch;" class="border-bottom" id="window-tabs">
            <h4>F-Chat</h4>
            <div class="btn" :class="'btn-' + (hasUpdate ? 'warning' : 'light')" @click="openMenu" id="settings">
                <i class="fa fa-cog"></i>
            </div>
            <ul class="nav nav-tabs" style="border-bottom:0;margin-bottom:-2px" ref="tabs">
                <li v-for="tab in tabs" :key="tab.view.id" class="nav-item" @auxclick="remove(tab)">
                    <a href="#" @click.prevent="show(tab)" class="nav-link"
                        :class="{active: tab === activeTab, hasNew: tab.hasNew && tab !== activeTab}">
                        <img v-if="tab.user" :src="'https://static.f-list.net/images/avatar/' + tab.user.toLowerCase() + '.png'"/>
                        {{tab.user || l('window.newTab')}}
                        <a href="#" class="btn" :aria-label="l('action.close')" style="margin-left:10px;padding:0;color:inherit"
                            @click.stop="remove(tab)"><i class="fa fa-times"></i>
                        </a>
                    </a>
                </li>
                <li v-show="canOpenTab" class="addTab nav-item" id="addTab">
                    <a href="#" @click.prevent="addTab" class="nav-link"><i class="fa fa-plus"></i></a>
                </li>
            </ul>
            <div style="flex:1;display:flex;justify-content:flex-end;-webkit-app-region:drag;margin-top:3px" class="btn-group"
                id="windowButtons">
                <i class="far fa-window-minimize btn btn-light" @click.stop="minimize"></i>
                <i class="far btn btn-light" :class="'fa-window-' + (isMaximized ? 'restore' : 'maximize')" @click="maximize"></i>
                <span class="btn btn-light" @click.stop="close">
                    <i class="fa fa-times fa-lg"></i>
                </span>
            </div>
        </div>
    </div>
</template>

<script lang="ts">
    import Sortable = require('sortablejs'); //tslint:disable-line:no-require-imports

    import * as electron from 'electron';
    import * as fs from 'fs';
    import * as path from 'path';
    import * as url from 'url';
    import Vue from 'vue';
    import Component from 'vue-class-component';
    import l from '../chat/localize';
    import {GeneralSettings} from './common';

    const browserWindow = electron.remote.getCurrentWindow();

    function getWindowBounds(): Electron.Rectangle {
        const bounds = browserWindow.getContentBounds();
        const height = document.body.offsetHeight;
        return {x: 0, y: height, width: bounds.width, height: bounds.height - height};
    }

    function destroyTab(tab: Tab): void {
        if(tab.user !== undefined) electron.ipcRenderer.send('disconnect', tab.user);
        tab.tray.destroy();
        tab.view.destroy();
        electron.ipcRenderer.send('tab-closed');
    }

    interface Tab {
        user: string | undefined,
        view: Electron.BrowserView
        hasNew: boolean
        tray: Electron.Tray
    }

    const trayIcon = path.join(__dirname, <string>require('./build/tray.png')); //tslint:disable-line:no-require-imports

    @Component
    export default class Window extends Vue {
        //tslint:disable:no-null-keyword
        settings!: GeneralSettings;
        tabs: Tab[] = [];
        activeTab: Tab | null = null;
        tabMap: {[key: number]: Tab} = {};
        isMaximized = browserWindow.isMaximized();
        canOpenTab = true;
        l = l;
        hasUpdate = false;
        platform = process.platform;

        mounted(): void {
            this.addTab();
            electron.ipcRenderer.on('settings', (_: Event, settings: GeneralSettings) => this.settings = settings);
            electron.ipcRenderer.on('allow-new-tabs', (_: Event, allow: boolean) => this.canOpenTab = allow);
            electron.ipcRenderer.on('open-tab', () => this.addTab());
            electron.ipcRenderer.on('update-available', (_: Event, available: boolean) => this.hasUpdate = available);
            electron.ipcRenderer.on('fix-logs', () => this.activeTab!.view.webContents.send('fix-logs'));
            electron.ipcRenderer.on('quit', () => this.destroyAllTabs());
            electron.ipcRenderer.on('connect', (_: Event, id: number, name: string) => {
                const tab = this.tabMap[id];
                tab.user = name;
                tab.tray.setToolTip(`${l('title')} - ${tab.user}`);
                const menu = this.createTrayMenu(tab);
                menu.unshift({label: tab.user, enabled: false}, {type: 'separator'});
                tab.tray.setContextMenu(electron.remote.Menu.buildFromTemplate(menu));
            });
            electron.ipcRenderer.on('disconnect', (_: Event, id: number) => {
                const tab = this.tabMap[id];
                if(tab.hasNew) {
                    tab.hasNew = false;
                    electron.ipcRenderer.send('has-new', this.tabs.reduce((cur, t) => cur || t.hasNew, false));
                }
                electron.ipcRenderer.send('disconnect', tab.user);
                tab.user = undefined;
                tab.tray.setToolTip(l('title'));
                tab.tray.setContextMenu(electron.remote.Menu.buildFromTemplate(this.createTrayMenu(tab)));
            });
            electron.ipcRenderer.on('has-new', (_: Event, id: number, hasNew: boolean) => {
                const tab = this.tabMap[id];
                tab.hasNew = hasNew;
                electron.ipcRenderer.send('has-new', this.tabs.reduce((cur, t) => cur || t.hasNew, false));
            });
            browserWindow.on('maximize', () => {
                this.isMaximized = true;
                this.activeTab!.view.setBounds(getWindowBounds());
            });
            browserWindow.on('unmaximize', () => {
                this.isMaximized = false;
                this.activeTab!.view.setBounds(getWindowBounds());
            });
            electron.ipcRenderer.on('switch-tab', (_: Event) => {
                const index = this.tabs.indexOf(this.activeTab!);
                this.show(this.tabs[index + 1 === this.tabs.length ? 0 : index + 1]);
            });
            electron.ipcRenderer.on('show-tab', (_: Event, id: number) => {
                this.show(this.tabMap[id]);
            });
            document.addEventListener('click', () => this.activeTab!.view.webContents.focus());
            window.addEventListener('focus', () => this.activeTab!.view.webContents.focus());

            Sortable.create(this.$refs['tabs'], {
                animation: 50,
                onEnd: (e: {oldIndex: number, newIndex: number}) => {
                    if(e.oldIndex === e.newIndex) return;
                    const tab = this.tabs.splice(e.oldIndex, 1)[0];
                    this.tabs.splice(e.newIndex, 0, tab);
                },
                onMove: (e: {related: HTMLElement}) => e.related.id !== 'addTab',
                filter: '.addTab'
            });

            window.onbeforeunload = () => {
                const isConnected = this.tabs.reduce((cur, tab) => cur || tab.user !== undefined, false);
                if(process.env.NODE_ENV !== 'production' || !isConnected) {
                    this.destroyAllTabs();
                    return;
                }
                if(!this.settings.closeToTray)
                    return setImmediate(() => {
                        if(confirm(l('chat.confirmLeave'))) {
                            this.destroyAllTabs();
                            browserWindow.close();
                        }
                    });
                browserWindow.hide();
                return false;
            };
        }

        destroyAllTabs(): void {
            browserWindow.setBrowserView(null!);
            this.tabs.forEach(destroyTab);
            this.tabs = [];
        }

        get styling(): string {
            try {
                return `<style>${fs.readFileSync(path.join(__dirname, `themes/${this.settings.theme}.css`))}</style>`;
            } catch(e) {
                if((<Error & {code: string}>e).code === 'ENOENT' && this.settings.theme !== 'default') {
                    this.settings.theme = 'default';
                    return this.styling;
                }
                throw e;
            }
        }

        trayClicked(tab: Tab): void {
            browserWindow.show();
            if(this.isMaximized) browserWindow.maximize();
            this.show(tab);
        }

        createTrayMenu(tab: Tab): Electron.MenuItemConstructorOptions[] {
            return [
                {label: l('action.open'), click: () => this.trayClicked(tab)},
                {label: l('action.quit'), click: () => this.remove(tab, false)}
            ];
        }

        addTab(): void {
            const tray = new electron.remote.Tray(trayIcon);
            tray.setToolTip(l('title'));
            tray.on('click', (_) => this.trayClicked(tab));
            const view = new electron.remote.BrowserView();
            view.setAutoResize({width: true, height: true});
            view.webContents.loadURL(url.format({
                pathname: path.join(__dirname, 'index.html'),
                protocol: 'file:',
                slashes: true,
                query: {settings: JSON.stringify(this.settings)}
            }));
            electron.ipcRenderer.send('tab-added', view.webContents.id);
            const tab = {active: false, view, user: undefined, hasNew: false, tray};
            tray.setContextMenu(electron.remote.Menu.buildFromTemplate(this.createTrayMenu(tab)));
            this.tabs.push(tab);
            this.tabMap[view.webContents.id] = tab;
            this.show(tab);
        }

        show(tab: Tab): void {
            this.activeTab = tab;
            browserWindow.setBrowserView(tab.view);
            tab.view.setBounds(getWindowBounds());
            tab.view.webContents.focus();
        }

        remove(tab: Tab, shouldConfirm: boolean = true): void {
            if(shouldConfirm && tab.user !== undefined && !confirm(l('chat.confirmLeave'))) return;
            this.tabs.splice(this.tabs.indexOf(tab), 1);
            electron.ipcRenderer.send('has-new', this.tabs.reduce((cur, t) => cur || t.hasNew, false));
            delete this.tabMap[tab.view.webContents.id];
            if(this.tabs.length === 0) {
                browserWindow.setBrowserView(null!);
                if(process.env.NODE_ENV === 'production') browserWindow.close();
            } else if(this.activeTab === tab) this.show(this.tabs[0]);
            destroyTab(tab);
        }

        minimize(): void {
            browserWindow.minimize();
        }

        maximize(): void {
            if(browserWindow.isMaximized()) browserWindow.unmaximize();
            else browserWindow.maximize();
        }

        close(): void {
            browserWindow.close();
        }

        openMenu(): void {
            electron.remote.Menu.getApplicationMenu()!.popup();
        }
    }
</script>

<style lang="scss">
    #window-tabs {
        user-select: none;
        .btn {
            border-radius: 0;
            padding: 2px 15px;
            display: flex;
            margin: 0px -1px -1px 0;
            align-items: center;
            -webkit-app-region: no-drag;
        }

        .btn-default {
            background: transparent;
        }

        li {
            height: 100%;
            a {
                display: flex;
                padding: 2px 10px;
                height: 100%;
                align-items: center;
                &:first-child {
                    border-top-left-radius: 0;
                }
            }

            img {
                height: 28px;
                margin: -5px 3px -5px -5px;
            }

            &.active {
                margin-bottom: -2px;
            }
        }

        h4 {
            margin: 0 10px;
            user-select: none;
            cursor: default;
            align-self: center;
            -webkit-app-region: drag;
        }

        .fa {
            line-height: inherit;
        }
    }

    #windowButtons .btn {
        margin: -4px -1px -1px 0;
        border-top: 0;
    }

    .platform-darwin {
        #windowButtons .btn, #settings {
            display: none;
        }

        #window-tabs {
            h4 {
                margin: 0 34px 0 77px;
            }

            .btn, li a {
                padding-top: 5px;
                padding-bottom: 5px;
            }
        }
    }
</style>