<template>
	<view class="base-wrapper py-safe" :style="[{ paddingTop: paddingTop + 'px' },Bglinear]">
		<slot></slot>
		<view class="tabbar-placeholder"></view>

		<!-- 全局固定客服联系按钮 -->
		<button v-if="showCustomerService" 
			class="customer-service-btn" 
			:class="{ dragging: isDragging }"
			:style="{ left: currentX + 'px', top: currentY + 'px' }"
			open-type="contact"
			@touchstart="onTouchStart"
			@touchmove="onTouchMove"
			@touchend="onTouchEnd"
			@click="handleCustomerClick"
			@contact="onContact"
			@getphonenumber="onGetPhoneNumber"
			@error="onError">
			<u-icon name="server-man" color="#fff" size="32"></u-icon>
		</button>
	</view>
</template>

<script>
	export default {
		name: 'BaseWrapper',
		props: {
			Bglinear: {
				type: [Object, String],
				default: () => ({})
			},
			showCustomerService: {
				type: Boolean,
				default: true // 默认显示客服按钮
			}
		},
		data() {
			return {
				paddingTop: 0,
				// 客服按钮拖动相关
				isDragging: false,
				startX: 0,
				startY: 0,
				currentX: 32, // 默认左边距32px
				currentY: 100, // 默认100px位置
				screenWidth: 375,
				screenHeight: 812,
				buttonSize: 100, // 按钮大小
				dragThreshold: 5 // 拖动阈值，超过这个距离才算拖动
			}
		},
		computed: {
		},
		mounted() {
			console.log(this.showCustomerService,'asd');
			let padding = 0;
			// 优先使用安全区信息
			const sysInfo = uni.getSystemInfoSync();
			if (sysInfo.safeAreaInsets && sysInfo.safeAreaInsets.top) {
				padding = sysInfo.safeAreaInsets.top;
			} else if (sysInfo.statusBarHeight) {
				padding = sysInfo.statusBarHeight;
			}
			this.paddingTop = padding;

			// 获取屏幕尺寸
			this.screenWidth = sysInfo.screenWidth || 375;
			this.screenHeight = sysInfo.screenHeight || 812;

			// 从本地存储读取客服按钮位置
			this.loadCustomerServicePosition();
		},
		methods: {
			// 从本地存储加载客服按钮位置
			loadCustomerServicePosition() {
				try {
					const position = uni.getStorageSync('customerServicePosition');
					if (position) {
						const { x, y } = JSON.parse(position);
						// 验证位置是否在有效范围内
						if (this.isValidPosition(x, y)) {
							this.currentX = x;
							this.currentY = y;
						}
					}
				} catch (error) {
					console.error('读取客服按钮位置失败:', error);
				}
			},

			// 保存客服按钮位置到本地存储
			saveCustomerServicePosition() {
				try {
					const position = {
						x: this.currentX,
						y: this.currentY
					};
					console.log(position)
					uni.setStorageSync('customerServicePosition', JSON.stringify(position));
				} catch (error) {
					console.error('保存客服按钮位置失败:', error);
				}
			},

			// 验证位置是否有效
			isValidPosition(x, y) {
				const buttonSize = 50; // 按钮大小50px
				const maxX = this.screenWidth - buttonSize;
				const minX = 0;
				const maxY = this.screenHeight - buttonSize;
				const minY = 0;

				return x >= minX && x <= maxX && y >= minY && y <= maxY;
			},

			// 触摸开始
			onTouchStart(e) {
				e.preventDefault();
				e.stopPropagation();
				
				this.isDragging = false;
				this.startX = e.touches[0].clientX;
				this.startY = e.touches[0].clientY;
			},

			// 触摸移动
			onTouchMove(e) {
				e.preventDefault();
				e.stopPropagation();
				
				const currentTouchX = e.touches[0].clientX;
				const currentTouchY = e.touches[0].clientY;
				console.log(currentTouchX,currentTouchY)
				const deltaX = Math.abs(currentTouchX - this.startX);
				const deltaY = Math.abs(currentTouchY - this.startY);
				
				// 超过阈值才开始拖动
				if (!this.isDragging && (deltaX > this.dragThreshold || deltaY > this.dragThreshold)) {
					this.isDragging = true;
				}
				
				if (this.isDragging) {
					// 直接计算新位置（像素）
					const moveX = currentTouchX - this.startX;
					const moveY = currentTouchY - this.startY;
					
					let newX = this.currentX + moveX;
					let newY = this.currentY + moveY;
					
					// 边界检测
					const buttonSize = 50; // 按钮大小50px
					const maxX = this.screenWidth - buttonSize;
					const minX = 0;
					const maxY = this.screenHeight - buttonSize;
					const minY = 0;
					
					newX = Math.max(minX, Math.min(maxX, newX));
					newY = Math.max(minY, Math.min(maxY, newY));
					
					// 实时更新位置
					this.currentX = newX;
					this.currentY = newY;
					
					// 更新起始位置为当前触摸位置
					this.startX = currentTouchX;
					this.startY = currentTouchY;
				}
			},

			// 触摸结束
			onTouchEnd(e) {
				e.preventDefault();
				e.stopPropagation();
				
				if (this.isDragging) {
					// 磁性吸附到边缘
					this.magneticAdsorption();
					
					// 拖动结束，最终保存位置
					this.saveCustomerServicePosition();
					
					// 延迟重置拖动状态，避免触发click事件
					setTimeout(() => {
						this.isDragging = false;
					}, 100);
				} else {
					this.isDragging = false;
				}
			},

			// 磁性吸附到边缘
			magneticAdsorption() {
				const buttonSize = 50; // 按钮大小50px
				const centerX = this.screenWidth / 2;
				
				// 如果在左半边，吸附到左边；如果在右半边，吸附到右边
				if (this.currentX < centerX) {
					this.currentX = 20; // 左边距20px
				} else {
					this.currentX = this.screenWidth - buttonSize - 20; // 右边距20px
				}
			},

			// 处理客服按钮点击
			handleCustomerClick(e) {
				// 如果刚拖动过，阻止点击事件
				if (this.isDragging) {
					e.preventDefault();
					e.stopPropagation();
					return false;
				}
				
				// #ifdef MP-WEIXIN
				// 微信小程序会自动处理open-type="contact"，不需要额外处理
				console.log('微信小程序客服会话将自动打开');
				// #endif
				
				// #ifndef MP-WEIXIN
				// 其他平台使用自定义客服功能
				this.contactCustomerService();
				// #endif
			},

			// 客服会话事件处理
			onContact(e) {
				console.log('进入客服会话:', e);
			},

			// 获取手机号事件处理（如果需要）
			onGetPhoneNumber(e) {
				console.log('获取手机号:', e);
			},

			// 错误处理
			onError(e) {
				console.error('按钮错误:', e);
				// 如果微信客服失败，回退到自定义客服
				this.contactCustomerService();
			},

			// 联系客服
			contactCustomerService() {
				uni.showActionSheet({
					itemList: ['在线客服', '电话咨询', '微信客服'],
					success: (res) => {
						switch (res.tapIndex) {
							case 0:
								// 在线客服
								this.openOnlineService();
								break;
							case 1:
								// 电话咨询
								this.callCustomerService();
								break;
							case 2:
								// 微信客服
								this.openWechatService();
								break;
						}
					},
					fail: (err) => {
						console.log('取消选择客服方式');
					}
				});
			},

			// 打开在线客服
			openOnlineService() {
				uni.showToast({
					title: '正在连接客服...',
					icon: 'loading',
					duration: 1500
				});

				// TODO: 这里可以跳转到在线客服页面
				setTimeout(() => {
					uni.showModal({
						title: '在线客服',
						content: '客服工作时间：09:00-18:00\n为您提供专业的产品咨询服务',
						showCancel: false,
						confirmText: '知道了',
						confirmColor: '#a6e22e'
					});
				}, 1500);
			},

			// 拨打客服电话
			callCustomerService() {
				const phoneNumber = '400-123-4567'; // 客服电话号码
				uni.showModal({
					title: '客服电话',
					content: `即将拨打客服热线：${phoneNumber}\n工作时间：09:00-18:00`,
					confirmText: '立即拨打',
					cancelText: '取消',
					confirmColor: '#a6e22e',
					success: (res) => {
						if (res.confirm) {
							uni.makePhoneCall({
								phoneNumber: phoneNumber,
								success: () => {
									console.log('拨打电话成功');
								},
								fail: (err) => {
									console.error('拨打电话失败:', err);
									uni.showToast({
										title: '拨打失败，请手动拨打',
										icon: 'none',
										duration: 2000
									});
								}
							});
						}
					}
				});
			},

			// 打开微信客服
			openWechatService() {
				uni.showModal({
					title: '微信客服',
					content: '请添加微信号：leibonongye\n或扫描二维码添加客服微信',
					showCancel: true,
					confirmText: '复制微信号',
					cancelText: '取消',
					confirmColor: '#a6e22e',
					success: (res) => {
						if (res.confirm) {
							uni.setClipboardData({
								data: 'leibonongye',
								success: () => {
									uni.showToast({
										title: '微信号已复制',
										icon: 'success',
										duration: 2000
									});
								}
							});
						}
					}
				});
			}
		}
	}
</script>

<style lang="scss" scoped>
	.base-wrapper {
		width: 100%;
		box-sizing: border-box;
		min-height: 100vh;
		height: auto;
		display: flex;
		flex-direction: column;
		position: relative;
	}

	.tabbar-placeholder {
		width: 100%;
		height: 160rpx;
		// height: 60rpx;
		min-height: env(safe-area-inset-bottom);
		pointer-events: none;
	}

	// 全局固定客服按钮
	.customer-service-btn {
		position: fixed;
		z-index: 9999;
		width: 50px;
		height: 50px;
		background: linear-gradient(135deg, #a6e22e 0%, #8db82a 100%);
		border-radius: 50%;
		display: flex;
		align-items: center;
		justify-content: center;
		box-shadow:
			0 12rpx 40rpx rgba(166, 226, 46, 0.3),
			0 4rpx 16rpx rgba(0, 0, 0, 0.1);
		cursor: pointer;
		user-select: none;
		-webkit-user-select: none;
		/* 初始位置由JavaScript设置，不在CSS中设置left/top */

		// 只在非拖动状态下显示动画
		&:not(.dragging) {
			animation: float-up-down 2s ease-in-out infinite;
		}

		&:active {
			box-shadow:
				0 8rpx 24rpx rgba(166, 226, 46, 0.4),
				0 2rpx 8rpx rgba(0, 0, 0, 0.15);
			animation-play-state: paused;
		}

		// 拖动状态样式
		&.dragging {
			animation: none;
			transform: translateY(-50%) scale(1.1);
			box-shadow:
				0 16rpx 48rpx rgba(166, 226, 46, 0.4),
				0 8rpx 24rpx rgba(0, 0, 0, 0.2);
		}

		// 悬浮光晕效果
		&::before {
			content: '';
			position: absolute;
			top: -8rpx;
			left: -8rpx;
			right: -8rpx;
			bottom: -8rpx;
			background: radial-gradient(circle, rgba(166, 226, 46, 0.2) 0%, transparent 70%);
			border-radius: 50%;
			opacity: 0;
			transition: opacity 0.3s ease;
			z-index: -1;
		}

		&:hover::before,
		&.dragging::before {
			opacity: 1;
		}
	}

	// 上下运动动画
	@keyframes float-up-down {

		0%,
		100% {
			transform: translateY(-50%) translateX(0);
		}

		50% {
			transform: translateY(-50%) translateX(0) translateY(-20rpx);
		}
	}

	// 响应式适配
	@media screen and (max-width: 750rpx) {
		.customer-service-btn {
			width: 80rpx;
			height: 80rpx;
			left: 24rpx;
		}
	}
</style>