<!--
@desc 弹幕消息系统，简单实现动画效果
@todo 究极版下一次大迭代=>消息动画直接击中武汉地理坐标 2020年1月31日19:08:26
-->
<template>
		<div class="barrage-module" :style="!isBarrageMode?{'z-index':'-1'}:{'z-index':'100'}">
				
				<div class="display-wrap">
						<ul class="ul">
								<li class="animate-to-left" :style="{left:leftAnimation[index]+'px',top:index*66+'px'}"
								    v-for="(item,index) in barrages">
										<img :src="item.avatarUrl" :alt="item.name||''">
										<span v-show="item.name">{{item.name}}：</span>
										<span :style="{color:item.color}">{{item.message||""}}</span>
								</li>
						</ul>
				</div>
				
				<div class="footer-input clear">
						<button class="input-button out-button" @click="onOutRoom">退出弹幕</button>
						<input class="input-text" v-model="inputContent" type="text" @keyup.enter="onEnterInput"
						       :disabled="!authObj.isAuth"
						       placeholder="请输入你要喊武汉加油的话！" maxlength="32">
						
						<button v-if="authObj.isAuth" class="input-button" @click="onEnterInput">发送</button>
						<a :href="authObj.oAuthUrl||''" target="_blank">
								<button v-if="!authObj.isAuth" class="input-button">
										授权
										<img style="width: 24px;
										     height: 24px;
										     top: 4px;
												 position: relative;"
										     src="../../assets/images/github-logo.png" alt="">
								</button>
						
						</a>
				</div>
		</div>
</template>

<script>
	import {emitSocket, onSocket} from "../../utils/socketIo";
	
	export default {
		name: "Barrage",
		props: {
			authObj: {
				type: Object
			},
			isBarrageMode: {
				type: Boolean,
				default() {
					return true
				}
			}
		},
		data() {
			return {
				clickTickTimestamp: [],
				barrageContent: [],
				inputContent: "武汉加油！",
				clearTimer: null,
				leftMoveTimer: null,
				width: window.innerWidth,
				leftAnimation: [],
				pageData: {
					size: 10,
					page: 1,
					count: 0
				}
			}
		},
		watch: {
			barrageContent: {
				handler(val) {
					if (val.length) {
						this.createdLeft(val.length)
					} else {
						clearInterval(this.leftMoveTimer)
					}
				},
				deep: true
			}
		},
		computed: {
			randomSetup() {
				let len = this.barrageContent.length || 0;
				let randomArray = [];
				let speedAra = [2, 3, 5, 8];
				for (let i = 0; i < len; i++) {
					randomArray.push(speedAra[Math.floor(Math.random() * 4)])
				}
				return randomArray
				
			},
			barrages() {
				return this.barrageContent.slice(0, 10)
			},
		},
		mounted() {
			onSocket.call(this, 'talk');
			onSocket.call(this, 'getTalk');
			this.getHistoryBarrage();
			this.clearTimer = setInterval(() => {
				this.clickTickTimestamp.shift()
			}, 3000);
		},
		destroyed() {
			clearInterval(this.clearTimer)
		},
		methods: {
			/**
			 * @desc 获取历史弹幕记录
			 * */
			getHistoryBarrage() {
				emitSocket('getTalk', this.pageData);
			},
			/**
			 * @desc 简单实现弹幕效果
			 * */
			createdLeft(len) {
				clearInterval(this.leftMoveTimer);
				this.leftMoveTimer = setInterval(() => {
					for (let i = 0; i < len; i++) {
						if (this.leftAnimation[i] === null || Number.isNaN(this.leftAnimation[i])) {
							this.leftAnimation[i] = this.width / 2 - this.randomSetup[i]
						} else {
							this.leftAnimation[i] = this.leftAnimation[i] - this.randomSetup[i]
						}
					}
					let maxVal = Math.max.apply(null, this.leftAnimation);
					if (maxVal < -(this.width / 1.5)) {
						for (let i = 0; i < len; i++) {
							this.leftAnimation[i] = this.width / 1.5;
						}
						this.barrageContent.splice(0, 10);
						// 下一批弹幕
						console.info('page,count=>', this.pageData);
						if (this.pageData.page + 1 <= this.pageData.count) {
							this.pageData.page++;
							this.$nextTick(() => {
								emitSocket('getTalk', this.pageData);
							})
						}
						
						clearInterval(this.leftMoveTimer);
					}
					if (!this.barrageContent.length) {
						clearInterval(this.leftMoveTimer);
					}
				}, 20)
			},
			onOutRoom() {
				this.$emit('emitBarrageHide', false)
			},
			onEnterInput() {
				if (this.clickTickTimestamp.length < 5) {
					this.clickTickTimestamp.splice(0, 0, new Date().getTime());
				}
				if (this.clickTickTimestamp.length > 4) {
					alert('太频繁，请稍后重试');
					return
				}
				if (!this.inputContent.trim()) {
					alert('内容为空，无法发送');
					return false
				}
				this.$nextTick(() => {
					this.inputContent = '武汉加油'
				});
				emitSocket('talk', this.inputContent);
			}
		}
	};
</script>

<style scoped lang="scss">
		
		.display-wrap {
				position: absolute;
				left: 50%;
				top: 50%;
				text-align: center;
				transform: translate(-50%, -50%);
				width: 100%;
				max-height: 100%;
				color: #fff;
		}
		
		.barrage-module {
				position: absolute;
				width: 100%;
				height: 100%;
				left: 0;
				top: 0;
				z-index: 100;
				background: rgba(255, 0, 0, 0.2);
		}
		
		.animate-to-left {
				display: inline-block;
				position: relative;
				font-size: 18px;
				line-height: 64px;
				font-weight: bold;
				left: 920px;
				
				img {
						border-radius: 50%;
						vertical-align: middle;
						margin-right: 10px;
						width: 60px;
						height: 60px;
				}
		}
		
		.footer-input {
				position: absolute;
				left: 50%;
				bottom: 20px;
				transform: translateX(-50%);
				width: 30%;
				min-width: 650px;
				
				a {
						text-decoration: none;
				}
				
				.input-text {
						display: block;
						float: left;
						width: 450px;
						height: 66px;
						padding: 8px 10px;
						font-size: 18px;
						color: #a4a9b0;
						background-image: none;
						box-shadow: none;
						border: 1px solid #ddd;
						box-sizing: border-box;
						-webkit-appearance: none;
						
						&:disabled {
								cursor: not-allowed;
						}
				}
				
				.input-button {
						display: block;
						line-height: 62px;
						width: 100px;
						font-size: 18px;
						background: #4caf50;
						border: 1px solid #4caf50;
						color: #fff;
						border-top-left-radius: 0;
						border-bottom-left-radius: 0;
						
				}
				
				.out-button {
						display: block;
						line-height: 62px;
						float: left;
						width: 100px;
						font-size: 18px;
						background: #FF8626;
						border: 1px solid #FF8626;
						color: #fff;
						border-radius: 8px 0 0 8px;
				}
		}
		
		.ul {
				position: relative;
				min-height: 660px;
		}
</style>
