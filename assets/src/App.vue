<template>
    <div id="app">
        <Input v-if="!terminal.show" v-model="searchString" icon="search" size="large" @on-change="handleSearch" placeholder="Please enter the filter key..." style="width: 400px" />
        <Table v-if="!terminal.show" :loading="devices.loading" :height="devices.height" :columns="devlistTitle" :data="devices.filtered" style="width: 100%"></Table>
        <div ref="terminal" class="terminal" v-if="terminal.show"></div>
        <Spin size="large" fix v-if="terminal.loading"></Spin>
        <Modal v-model="contextMenuVisible" width="21">
             <Menu theme="light" @on-select="handleContextMenu">
                <MenuItem name="upfile">Upload file to device</MenuItem>
                <MenuItem name="downfile">Download file from device</MenuItem>
                <MenuItem name="increasefontsize">Increase font size</MenuItem>
                <MenuItem name="decreasefontsize">Decrease font size</MenuItem>
             </Menu>
             <div slot="footer"></div>
        </Modal>
        <Modal v-model="upfile.modal" width="360" :mask-closable="false" @on-cancel="cancelUpfile">
            <p slot="header"><span>Upload file to device</span></p>
            <Upload :before-upload="beforeUpload" action="">
                <Button type="ghost" icon="Upload">Select the file to upload</Button>
            </Upload>
            <Progress v-if="upfile.file !== null" :percent="upfile.percent"></Progress>
            <div v-if="upfile.file !== null">The file "{{upfile.file.name}}" will be saved in the "/tmp/" directory of your device.</div>
            <div slot="footer">
                <Button type="primary" size="large" long :loading="upfile.loading" @click="doUpload">{{upfile.loading ? 'Uploading' : 'Click to upload'}}</Button>
            </div>
        </Modal>
        <Modal v-model="downfile.modal" width="700" :mask-closable="false">
            <p slot="header"><span>Download file from device</span></p>
            <Tag>{{downfile.pathname}}</Tag>
            <Table :loading="downfile.loading" v-if="!downfile.downing" :columns="filelistTitle" height="400" :data="downfile.filelist" @on-row-dblclick="filelistDblclick"></Table>
            <Progress v-if="downfile.downing" :percent="downfile.percent"></Progress>
            <div slot="footer"></div>
        </Modal>
    </div>
</template>

<script>

import * as Socket from 'simple-websocket';
import { Terminal } from 'xterm'
import 'xterm/lib/xterm.css'
import * as fit from 'xterm/lib/addons/fit/fit';
import axios from 'axios'
import * as rtty from './rtty'

Terminal.applyAddon(fit);

export default {
    data() {
        return {
            searchString: '',
            contextMenuVisible: false,
            terminal: {loading: false, show: false, term: null, recvCnt: 0},
            devices: {loading: true, height: document.body.offsetHeight - 20, list: [], filtered: []},
            upfile: {modal: false, file: null, step: 2048, pos: 0, canceled: false, percent: 0},
            downfile: {modal: false, loading: true, path: ['/'], pathname: '/', filelist: [], downing: false, percent: 0},
            ws: null,
            sid: '',
            username: '',
            password: '',
            devId: '',
            devlistTitle: [
                {
                    title: 'ID',
                    key: 'id',
                    sortType: 'asc',
                    sortable: true
                }, {
                    title: 'Uptime',
                    key: 'uptime',
                    sortable: true,
                    render: (h, params) => {
                        return h('div', '%t'.format(params.row.uptime));
                    }
                }, {
                    title: 'Description',
                    key: 'description'
                }, {
                    width: 150,
                    align: 'center',
                    render: (h, params) => {
                        return h('Button', {
                            props: { type: 'primary' },
                            on: {
                                click: () => {
                                    this.terminal.loading = true;
                                    this.terminal.show = true;
                                    this.devId = params.row.id;
                                    window.setTimeout(this.login, 200);
                                }
                            }
                        }, 'Connect');
                    }
                }
            ],
            filelistTitle: [
                {
                    title: 'Name',
                    key: 'name',
                    render: (h, params) => {
                        if (params.row.dir)
                            return h('div', [
                                h('Icon', {props: {type: 'folder', color: '#FFE793', size: 20}}),
                                h('strong', ' ' + params.row.name)
                            ]);
                        else
                            return params.row.name;
                    }
                }, {
                    title: 'Size',
                    key: 'size',
                    sortable: true,
                    render: (h, params) => params.row.size && '%1024mB'.format(params.row.size)
                }, {
                    title: 'modification',
                    key: 'mtim',
                    sortable: true,
                    render: (h, params) => {
                        if (params.row.mtim)
                            return new Date(params.row.mtim * 1000).toLocaleString();
                    }
                }
            ]
        }
    },
    methods: {
        handleSearch() {
            this.devices.filtered = this.devices.list.filter(d => {
                return d.id.indexOf(this.searchString) > -1 || d.description.indexOf(this.searchString) > -1;
            });
        },
        handleContextMenu(name) {
            this.contextMenuVisible = false;
            let changeFontSize = 0;
            if (name == 'upfile') {
                this.upfile = {modal: true, loading: false, file: null, step: 2048, pos: 0, canceled: false, percent: 0};
            } else if (name == 'downfile') {
                this.downfile = {modal: true, loading: true, path: [], pathname: '/', filelist: [], downing: false, percent: 0};

                let pkt = rtty.newPacket(rtty.RTTY_PACKET_DOWNFILE, {sid: this.sid});
                this.ws.send(pkt);
            } else if (name == 'increasefontsize') {
                changeFontSize = 1;
            } else if (name == 'decreasefontsize') {
                changeFontSize = -1;
            }

            window.setTimeout(() => {
                let size = this.terminal.term.getOption('fontSize');
                this.terminal.term.setOption('fontSize', size + changeFontSize);
                this.terminal.term.fit();
                this.terminal.term.focus();
            }, 1);
        },
        beforeUpload (file) {
            this.upfile.file = file;
            return false;
        },
        readFile(fr) {
            var blob = this.upfile.file.slice(this.upfile.pos, this.upfile.pos + this.upfile.step);
            fr.readAsArrayBuffer(blob);
        },
        doUpload () {
            if (!this.upfile.file) {
                this.$Message.error('Please select file to upload.');
                return;
            }

            this.upfile.loading = true;
            
            var fr = new FileReader();
            fr.onload = (e) => {
                if (this.upfile.canceled)
                    return;

                let pkt = rtty.newPacket(rtty.RTTY_PACKET_UPFILE, {sid: this.sid, code: 1, data: fr.result});
                this.ws.send(pkt);
                this.upfile.pos += e.loaded;
                this.upfile.percent = Math.round(this.upfile.pos / this.upfile.file.size * 100)

                if (this.upfile.pos < this.upfile.file.size) {
                    /* Control the client read speed based on the current buffer and server */
                    if (this.ws.bufferedAmount > this.upfile.pos * 10 || this.ratelimit) {
                        this.ratelimit = false;

                        setTimeout(() => {
                            this.readFile(fr);
                        }, 100);
                    } else {
                        this.readFile(fr);
                    }
                } else {
                    this.upfile.modal = false;
                    this.$Message.info("Upload success");
                }
            };

            let pkt = rtty.newPacket(rtty.RTTY_PACKET_UPFILE, {sid: this.sid, name: this.upfile.file.name, size: this.upfile.file.size, code: 0});
            this.ws.send(pkt);
            this.readFile(fr);
        },
        cancelUpfile() {
            if (!this.upfile.loading)
                return;
            this.upfile.canceled = true;
            this.$Message.info("Upload canceled");
            let pkt = rtty.newPacket(rtty.RTTY_PACKET_UPFILE, {sid: this.sid, code: 2});
            this.ws.send(pkt);
        },
        filelistDblclick(row, index) {
            let attr = {sid: this.sid};

            if (row.name == '..') {
                if (this.downfile.path.length < 1)
                    return;
                this.downfile.path.pop();
            } else {
                this.downfile.path.push(row.name);
            }

            this.downfile.pathname = '/' + this.downfile.path.join('/');

            if (row.dir) {
                this.downfile.loading = true;
                if (!this.downfile.pathname.endsWith('/'))
                    this.downfile.pathname = this.downfile.pathname + '/';
            } else {
                this.downfile.received = 0;
                this.downfile.size = row.size;
                this.downfile.downing = true;
            }

            attr.name = this.downfile.pathname;
            let pkt = rtty.newPacket(rtty.RTTY_PACKET_DOWNFILE, attr);
            this.ws.send(pkt);
        },

        cancelDownfile() {
            let pkt = rtty.newPacket(rtty.RTTY_PACKET_DOWNFILE, {sid: this.sid, code: 1});
            this.ws.send(pkt);
            this.$Message.info("Download canceled");
        },
        getQueryString(name) {
            var reg = new RegExp("(^|&)" + name + "=([^&]*)(&|$)", "i");
            var r = window.location.search.substr(1).match(reg);
            if (r != null)
                return unescape(r[2]);
            return null;
        },
        logout(ws, term) {
            this.terminal.show = false;

            if (ws)
                ws.destroy();
            if (term)
                term.destroy();
        },
        login() {
            var term = new Terminal({
                cursorBlink: true,
                fontSize: 16
            });
            term.open(this.$refs['terminal']);
            term.fit();
            term.focus();
            this.terminal.term = term;

            var protocol = 'ws://';
            if (location.protocol == 'https:')
                protocol = 'wss://';

            var ws = new Socket(protocol + location.host + '/ws?devid=' + this.devId);
            ws.on('connect', () => {
                ws.on('data', (data) => {
                    let pkt = rtty.parsePacket(data);

                    if (pkt.typ == rtty.RTTY_PACKET_LOGINACK) {
                        this.terminal.loading = false;

                        if (pkt.code != 0) {
                            this.$Message.error('Device offline');
                            this.logout(null, term);
                            return;
                        }
                        this.ws = ws;
                        this.sid = pkt.sid;
                        term.on('data', (data) => {
                            let pkt = rtty.newPacket(rtty.RTTY_PACKET_TTY, {sid: this.sid, data: Buffer.from(data)});
                            ws.send(pkt);
                        });

                        term.attachCustomKeyEventHandler((e) => {
                            if (e.type == 'keydown') {
                                if (e.ctrlKey && e.shiftKey) {
                                    if (e.key == 'F') {
                                        this.contextMenuVisible = true;
                                    }
                                }
                            }
                        });
                    } else if (pkt.typ == rtty.RTTY_PACKET_TTY) {
                        this.terminal.recvCnt++;
                        var data = pkt.data.toString();
                        if (this.terminal.recvCnt < 4) {
                            if (data.match('login:') && this.username != '') {
                                let pkt = rtty.newPacket(rtty.RTTY_PACKET_TTY, {sid: this.sid, data: this.username + '\n'});
                                ws.send(pkt);
                                return;
                            }

                            if (data.match('Password:') && this.password != '') {
                                let pkt = rtty.newPacket(rtty.RTTY_PACKET_TTY, {sid: this.sid, data: this.password + '\n'});
                                ws.send(pkt);
                                return;
                            }
                        }
                        term.write(data);
                    } else if (pkt.typ == rtty.RTTY_PACKET_DOWNFILE) {
                        let code = pkt.code;
                        if (code == 0) {
                            this.downfile.loading = false;
                            this.downfile.filelist = JSON.parse(pkt.data.toString());
                        }
                        else if (code == 1) {
                            if (!this.downfile.data)
                                this.downfile.data = new Blob([pkt.data]);
                            else
                                this.downfile.data = new Blob([this.downfile.data, pkt.data]);
                            this.downfile.received += pkt.data.byteLength;
                            this.downfile.percent = Math.round(this.downfile.received / this.downfile.size * 100);
                        } else if (code == 2) {
                            let url = URL.createObjectURL(this.downfile.data);
                            let a = document.createElement('a');
                            a.download = this.downfile.pathname;
                            a.href = url;
                            a.click();
                            URL.revokeObjectURL(url);
                            this.downfile.modal = false;
                            this.$Message.info("Download Finish");
                        }
                    } else if (pkt.typ == rtty.RTTY_PACKET_UPFILE) {
                        if (pkt.code == 5) {
                            /* Need reduce the sending rate */
                            this.ratelimit = true;
                        }
                    }
                });

                ws.on('error', ()=> {
                    this.logout(null, term);
                });

                ws.on('close', ()=> {
                    this.logout(null, term);
                });
            })
        }
    },
    mounted() {
        var devId = this.getQueryString('id');
        var username = this.getQueryString('username');
        var password = this.getQueryString('password');

        if (username)
            this.username = username;
        if (password)
            this.password = password;

        if (devId) {
            this.terminal.loading = true;
            this.terminal.show = true;
            this.devId = devId;
            window.setTimeout(this.login, 200);
        }

        window.setInterval(() => {
            if (this.terminal.show)
                return;
            axios.get('/devs').then(res => {
                this.devices.loading = false;
                this.devices.list = res.data;
                this.handleSearch();
            });
        }, 2000);

        window.addEventListener("resize", () => {
            this.devices.height = document.body.offsetHeight - 20;
            if (this.terminal.show) {
                this.terminal.term.fit();
            }
        });
    }
}
</script>

<style>
    html, body {
		width: 100%;
	    height: 99%;
        background-color: #555;
    }

	#app {
	    width: 100%;
	    height: 100%;
        background-color: #555;
    }

    .terminal {
        height: 100%;
        margin-left: 5px;
        margin-top: 10px;
    }
</style>
