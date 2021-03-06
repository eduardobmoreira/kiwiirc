<template>
    <div class="kiwi-startbnc" :class="[closing ? 'kiwi-startbnc--closing' : '']">
        <div class="kiwi-startbnc-section kiwi-startbnc-section-connection">
            <h2 v-html="greetingText"></h2>

            <div class="kiwi-startbnc-status">{{statusMessage}}</div>

            <form v-on:submit.prevent="startUp" class="kiwi-startbnc-form">
                <label>
                    <span>{{$t('username')}}</span>
                    <input type="text" v-model="username" :disabled="loading" />
                </label>
                <label>
                    <span>{{$t('password')}}</span>
                    <input type="password" v-model="password" :disabled="loading" />
                </label>

                <button type="submit" class="u-button u-button-primary u-submit" :disabled="loading || !username || !password">
                    <span v-if="!loading" v-html="buttonText"></span>
                    <i v-else class="fa fa-spinner fa-spin" aria-hidden="true"></i>
                </button>
            </form>
        </div>

        <div class="kiwi-startbnc-section kiwi-startbnc-section-info" :style="infoStyle">
            <div class="kiwi-startbnc-section-info-content" v-if="infoContent" v-html="infoContent">
                <h3>Take your conversations anywhere.</h3>
                <h4>IRC? No problem.</h4>

                <p>IRC has a history of being difficult, lets makes it usable.</p>
                <p>Just log in and get going.</p>
            </div>
        </div>
    </div>
</template>

<script>

import _ from 'lodash';
import state from 'src/libs/state';
import Logger from 'src/libs/Logger';

let log = Logger.namespace('Startup/kiwiBnc');

export default {
    created: function created() {
        window.t = this;
    },
    data: function data() {
        return {
            closing: false,
            loading: false,
            username: '',
            password: '',
            statusMessage: '',
        };
    },
    computed: {
        greetingText: function greetingText() {
            let greeting = state.settings.startupOptions.greetingText;
            return typeof greeting === 'string' ?
                greeting :
                this.$t('start_greeting');
        },
        buttonText: function buttonText() {
            if (this.loading) {
                return '';
            }

            let greeting = state.settings.startupOptions.buttonText;
            return typeof greeting === 'string' ?
                greeting :
                this.$t('start_button');
        },
        infoStyle: function infoStyle() {
            let style = {};
            let options = state.settings.startupOptions;

            if (options.infoBackground) {
                style['background-image'] = `url(${options.infoBackground})`;
            } else {
                style['background-color'] = '#333333';
            }

            return style;
        },
        infoContent: function infoContent() {
            return state.settings.startupOptions.infoContent || '';
        },
    },
    methods: {
        close: function close() {
            this.closing = true;
            this.$el.addEventListener('transitionend', (event) => {
                state.persistence.watchStateForChanges();
                this.$emit('start');
            }, false);
        },
        startUp: async function startUp() {
            this.statusMessage = this.$t('logging_in');
            this.loading = true;

            let bncnet = this.getBncNetwork();

            let cleanUpEvents = () => {
                bncnet.ircClient.off('registered', onRegistered);
                bncnet.ircClient.off('irc error', onError);
                bncnet.ircClient.off('close', onClose);
            };

            let onRegistered = async () => {
                cleanUpEvents();

                let bncNetworks = await bncnet.ircClient.bnc.getNetworks();
                for (let network of bncNetworks) {
                    network.buffers = [];
                    try {
                        let buffers = await bncnet.ircClient.bnc.getBuffers(network.name);
                        network.buffers = buffers;
                    } catch (err) {
                        // Log the error here or something
                        log.error(err);
                    }

                    this.addNetworkToState(network);
                }

                this.monitorNetworkChanges(bncnet, bncNetworks);
                this.close();
            };

            let onError = (event) => {
                cleanUpEvents();
                this.statusMessage = this.$t('invalid_login');
                this.loading = false;
            };

            let onClose = (event) => {
                cleanUpEvents();
                this.statusMessage = this.$t('invalid_login');
                this.loading = false;
            };

            bncnet.ircClient.once('registered', onRegistered);
            bncnet.ircClient.once('irc error', onError);
            bncnet.ircClient.once('close', onClose);
            bncnet.ircClient.connect();
        },

        getBncNetwork: function getBncNetwork() {
            let bnc = state.setting('bnc');

            if (bnc.network) {
                bnc.username = this.username;
                bnc.password = this.password;
                return bnc.network;
            }

            let options = state.settings.startupOptions;

            // Indicate that all our connections will be going through a BNC
            bnc.active = true;
            bnc.server = options.server || '';
            bnc.port = options.port || 6667;
            bnc.tls = !!options.tls;
            bnc.username = this.username;
            bnc.password = this.password;

            let bncnet = state.addNetwork('bnccontrol', this.username, {
                server: bnc.server,
                port: bnc.port,
                tls: bnc.tls,
            });

            bnc.network = bncnet;

            return bncnet;
        },

        addNetworkToState: function addNetworkToState(network) {
            // Expects network to be in the format of:
            //  {
            //  "buffers":[{"channel":"1","name":"#prawnsalad","joined":"1"}],
            //  "name":"freenode",
            //  "channel":"1",
            //  "connected":"1",
            //  "host":"irc.freenode.net",
            //  "port":"6667",
            //  "tls":"0",
            //  "nick":"notprawn99829"
            //  },
            let net = state.addNetwork(network.name, network.nick, {
                server: network.host,
                port: network.port,
                tls: network.tls,
                password: network.password,
                bncname: network.name,
                username: network.user,
            });

            network.buffers.forEach(buffer => {
                let newBuffer = state.addBuffer(net.id, buffer.name);
                if (buffer.joined) {
                    newBuffer.enabled = true;
                }
                if (buffer.seen) {
                    newBuffer.last_read = (new Date(buffer.seen)).getTime();
                }
            });
        },

        monitorNetworkChanges: function monitorNetworkChanges(bncNet, bncNetworks) {
            let existingNets = Object.create(null);
            function rememberNetworks() {
                state.networks.forEach(network => {
                    if (!network.connection.bncname) {
                        return;
                    }

                    existingNets[network.connection.bncname] = {
                        name: network.connection.bncname,
                        host: network.connection.server,
                        port: network.connection.port,
                        tls: network.connection.tls,
                        password: network.connection.password,
                        nick: network.nick,
                        username: network.username,
                    };
                });
            }

            rememberNetworks();

            let saveState = newVal => {
                state.networks.forEach(network => {
                    // Only deal with BNC networks
                    if (network.name === 'bnccontrol') {
                        return;
                    }

                    let bncName = network.connection.bncname;
                    let current = existingNets[bncName] || {};
                    let tags = {};

                    if (network.connection.server !== current.host) {
                        tags.host = network.connection.server;
                    }
                    if (network.connection.port !== current.port) {
                        tags.port = network.connection.port;
                    }
                    if (network.connection.tls !== current.tls) {
                        tags.tls = network.connection.tls;
                    }
                    if (network.connection.password !== current.password) {
                        tags.password = network.connection.password;
                    }
                    if (network.nick !== current.nick) {
                        tags.nick = network.nick;
                    }
                    if (network.username !== current.username) {
                        tags.user = network.username;
                    }

                    // A newly added network would not have a current name property set yet.
                    // Only save the network if we've entered a host.
                    if (!current.name && tags.host) {
                        network.connection.bncname = network.name;
                        bncNet.ircClient.bnc.addNetwork(
                            network.name,
                            tags.host,
                            tags.port,
                            tags.tls,
                            tags.nick,
                            tags.user,
                            tags.password,
                        );
                    } else if (current.name) {
                        bncNet.ircClient.bnc.saveNetwork(bncName, tags);
                    }
                });

                rememberNetworks();
            };

            let debouncedSaveState = _.debounce(saveState, 2000);

            state.$watch('networks', debouncedSaveState, { deep: true });

            // Just before we connect to a network, make sure the BNC is sabed and connected to
            // it or at least trying to connect.
            state.$on('network.connecting', event => {
                saveState();

                let netName = event.network.connection.bncname;
                let networkFromBnc = _.find(bncNetworks, { name: netName });
                if (networkFromBnc && !networkFromBnc.connected) {
                    bncNet.ircClient.raw('BOUNCER connect ' + netName);
                }
            });

            // Very hacky until we have network name renaming on the bnc. When a new network
            // is added, change the name to the next available network name.
            state.$on('network.new', event => {
                let currentNum = 1;
                let existingNet = true;
                while (existingNet) {
                    existingNet = _.find(state.networks, { name: 'Network' + currentNum });
                    if (!existingNet) {
                        event.network.name = 'Network' + currentNum;
                    }

                    currentNum++;
                }
            });

            state.$on('network.removed', event => {
                bncNet.ircClient.bnc.removeNetwork(event.network.connection.bncname);
            });

            state.$on('buffer.close', event => {
                let buffer = event.buffer;
                let network = event.buffer.getNetwork();
                let bncName = network.connection.bncname;

                if (bncName) {
                    bncNet.ircClient.bnc.closeBuffer(bncName, buffer.name);
                }
            });
        },
    },
};
</script>

<style>

.kiwi-startbnc {
    text-align: center;
    height: 100%;
}

.kiwi-startbnc-section {
    position: absolute;
    top: 0;
    bottom: 0;
    width: 50%;
    padding: 1em;
    box-sizing: border-box;
    transition: right 0.3s, left 0.3s;
    overflow-y: auto;
}
.kiwi-startbnc-section-info {
    right: 0;
    border: 0 solid #86b32d;
    border-left-width: 5px;
    background-size: cover;
    color: #fff;
    background-position: bottom;
    display: flex;
    align-items: center;
    justify-content: center;
    min-height: 100%;
}
.kiwi-startbnc-section-info-content {
    background: rgba(255, 255, 255, 0.74);
    margin: 3em;
    color: #1b1b1b;
    font-size: 1.5em;
    padding: 1em;
    line-height: 1.6em;
}


.kiwi-startbnc-section-connection {
    left: 0;
    margin-top: 3em;
    font-size: 1.2em;
}

.kiwi-startbnc-section-connection label {
    text-align: left;
    display: inline-block;
    margin-bottom: 1.5em;
}
.kiwi-startbnc-section-connection input {
    font-size: 1em;
    margin-top: 5px;
    padding: 0.3em 1em;
    width: 100%;
    box-sizing: border-box;
}
.kiwi-startbnc-status {
    margin: 1em 0;
    overflow: hidden;
    max-height: 40px;
    transition: max-height 0.2s;
}
.kiwi-startbnc-status:empty {
    background: red;
    max-height: 0;
}
.kiwi-startbnc-start {
    font-size: 1.1em;
    cursor: pointer;
}
.kiwi-startbnc-form {
    width: 300px;
    margin: 2em auto;
}
.kiwi-startbnc-server-types {
    font-size: 0.9em;
}
.kiwi-startbnc-server-types a {
    margin: 0 1em;
}

.kiwi-startbnc--closing .kiwi-startbnc-section-connection {
    left: -50%;
}
.kiwi-startbnc--closing .kiwi-startbnc-section-info {
    right: -50%;
}

/** Smaller screen...**/
@media screen and (max-width: 850px) {
    .kiwi-startbnc {
        font-size: 0.9em;
    }

    .kiwi-startbnc-section-connection {
        margin-top: 1em;
    }
    .kiwi-startbnc-section-info-content {
        margin: 1em;
    }
}

/** Even smaller screen.. probably phones **/
@media screen and (max-width: 750px) {
    .kiwi-startbnc {
        font-size: 0.9em;
        overflow-y: auto;
    }

    .kiwi-startbnc-section {
        left: 0;
        width: 100%;
        right: auto;
        position: relative;
    }

    .kiwi-startbnc-section-info {
        border-width: 5px 0 0 0;
    }
    .kiwi-startbnc-section-info-content {
        margin: 0.5em;
    }

    /** Closing - the wiping away of the screen **/
    .kiwi-startbnc--closing .kiwi-startbnc-section-connection {
        left: -100%;
    }
    .kiwi-startbnc--closing .kiwi-startbnc-section-info {
        left: -100%;
    }
}
</style>
