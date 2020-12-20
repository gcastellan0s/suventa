<style>
	@import '~assets/vendor/css/bootstrap-material.css';
	@import 'https://fonts.googleapis.com/css?family=Roboto:300,300i,400,400i,500,500i,700,700i,900';
	@import '~assets/vendor/fonts/fontawesome.css';
	@import '~assets/vendor/css/bootstrap-material.css';
	@import '~assets/vendor/css/appwork-material.css';
	@import '~assets/vendor/css/theme-corporate-material.css';
	@import '~assets/vendor/css/colors-material.css';
	@import '~assets/vendor/css/uikit.css';
	@import '~assets/vendor/css/pages/authentication.css';
</style>

<template>
	<div>
        <div v-if="$route.path === '/'">
            <div class="authentication-wrapper authentication-1">
                <div class="authentication-inner" style="max-width: 400px;">
                    <div class="d-flex justify-content-center align-items-center mb-3">
                        <img src="~assets/img/logoBG.png" style="max-width: 300px;"/>
                    </div>
                    <form>
                        <p>
                            Es posible que no tengas conexión a internet, porfavor espera un momento y reinicia la aplicación. :(
                        </p>
                        <div class="form-group">
                            <button type="button" class="btn btn-primary btn-block" v-on:click="location.reload()">{{REINICIAR}}</button>
                        </div>
                    </form>
                </div>
            </div>
        </div>
        <div v-if="$route.path === '/getCode'">
            <div class="authentication-wrapper authentication-1 px-4">
                <div class="authentication-inner py-5">
                    <div class="d-flex justify-content-center align-items-center mb-3">
                        <img src="~assets/img/logoBG.png" style="max-width: 300px;"/>
                    </div>
                    <form class="my-3" v-on:submit.prevent="getCode(code)">
                        <div class="form-group">
                            <label class="form-label d-flex justify-content-between align-items-end">
                                CÓDIGO DE EMPRESA
                            </label>
                            <input type="text" class="form-control" v-model="code" required/>
                            <small>Solo se pide una sola vez.</small>
                        </div>
                        <div class="form-group">
                            <button type="submit" class="btn btn-primary btn-block">ENTRAR</button>
                        </div>
                    </form>
                </div>
            </div>
        </div>
	</div>
</template>

<script>

import EscPosEncoder from 'esc-pos-encoder';
const encoder = new EscPosEncoder();

import VueCookies from 'vue-cookies'
import swal from 'sweetalert';
import Vue from 'vue'
import axios from 'axios'
import VueAxios from 'vue-axios'
Vue.use(VueAxios, axios)
const ax = axios.create({
    xsrfCookieName: 'csrftoken',
    xsrfHeaderName: 'X-CSRFToken',
    baseURL: 'http://mirari.xyz',
})
const qs = require('qs');
const variables = require('assets/js/variables.js');
var socketId = null;
var printerIP = variables.printerIP;
var printerPort = variables.printerPort
var localIP = null
var ref = null
var user = variables.user

var PublicSocket = require('socket.io-client')('http://50.18.229.242:8002');
PublicSocket.emit('room', variables.organizationCode);

let logo = 'img/logoBG.png'

var times = 1

export default {
	name: 'MyLayout',
	data () {
		return {
			leftDrawerOpen: false,
			REINICIAR: 'REINICIAR APLICACIÓN',
			url: '',
			code: '',
		}
	},
	created:function(){
		self=this
		if ($cookies.get('code')) self.code = $cookies.get('code')
		if ($cookies.get('url')) self.url = $cookies.get('url')
	},
	mounted: function () {
		if(!self.code || !self.url) self.$router.push('/getCode')
		else self.goIndex()
		PublicSocket.on('Recibe_Event', (event, data) => {
      console.log('Recibe_Event', event, data)
			if (user == data.user){
				printerIP = variables.printerIP
				if (event == 'Print') {
					this.createSocketPrint(event, data.ticketFormat)
				}
			}
		});
	},
	methods: {
		goIndex:((get="") => {
			self.$router.push('/');
			if(!self.code || !self.url){
				self.$router.push('/getCode');
			}
			else{
				if(ref==null){
					ref = cordova.InAppBrowser.open('http://' + self.url + get, '_blank', 'location=no,toolbar=no')
					ref.addEventListener('loaderror', function(event){ref.close()});
					ref.addEventListener("loadstart", (event) => {});
				}else{
					ref.show();
				}
			}
		}),
		getCode:((code) => {
			ax.post('/api/mirari/mirari/Organization/?api=getCodeOrganization', qs.stringify({code}))
			.then((json) => {
				$cookies.set('code', json.data.organization.code, "3y")
				$cookies.set('url', json.data.organization.sites.domain, "3y")
				location.reload()
			}).catch((error) => {
				Swal.fire('Ooops!', error.response.data.message, 'error')
			})
		}),
		createSocketPrint:( (event, data) => {
			chrome.sockets.tcp.create( function (createInfo) {
				chrome.sockets.tcp.connect(
					createInfo.socketId,
					printerIP,
					printerPort ? printerPort : 9100,
					function (result) {
						if (!result) {
							socketId = createInfo.socketId;
							self.makeTicket(event, data)
						} else {
							socketId = null;
						}
					}
				);
			});
		}),
		makeTicket:( (event, data) => {
			(async () => {
				times = 1
				await '.'.repeat(data[0][1]).split('').forEach(x => {
					let img = new Image();
					img.src = logo;
					img.onload = function() {
						let content = encoder
						content.initialize()
						content.codepage('cp437')
						data.forEach(t => {
							if (t[0] == 'set') {
								if (t[1][0] == 'CT') content.align('center')
								else if (t[1][0] == 'LT') content.align('left')
								else if (t[1][0] == 'RT') content.align('right')
								if (t[1][3] == 1 || t[1][4] == 1) content.raw([ 0x1b, 0x21, 0x00 ])
								else if (t[1][3] == 1 || t[1][4] == 2) content.raw([ 0x1b, 0x21, 0x20 ])
								else if (t[1][3] == 2 || t[1][4] == 1) content.raw([ 0x1b, 0x21, 0x10 ])
								else if (t[1][3] == 2 || t[1][4] == 2) content.raw([ 0x1b, 0x21, 0x30 ])
							}
							if (t[0] == 'text') {
								content.line(t[1])
							}
							if (t[0] == 'qr') {
								content.qrcode(t[1])
								content.newline()
								content.newline()
								content.newline()
								content.newline()
								content.raw([ 0x1d, 0x56, 0x00 ])
								setTimeout(
									function(){
										self.printTicket(socketId, content.encode())
									}, 1000 * times-1
								);
								times += 1
							}
							if (t[0] == 'cut'){
								content.newline()
								content.newline()
								content.newline()
                content.newline()
                content.newline()
								content.newline()
								content.raw([ 0x1d, 0x56, 0x00 ])
								content.raw([ 0x1b, 0x70, 0x00 ])
								setTimeout(
									function(){
										self.printTicket(socketId, content.encode())
									}, 1000 * times-1
								);
								times += 1
							}
						});

					}
				});
			})();
		}),
		printTicket:( (socketId, content) => {
			if (socketId == null) {
				return;
			}
			(async () => {
				await chrome.sockets.tcp.send(socketId, content.buffer, function (result) {
					chrome.sockets.tcp.disconnect(socketId, function (message) {
						console.log(message)
					})
				});
			})()
			return;
		}),
	},
}
</script>
