<template>
	<BaseContainer>
		<view class="notice-detail-container">
			<!-- 内容区域 -->
			<view class="content-area">
				<!-- 公告标题和信息 -->
				<view class="notice-header" v-if="noticeData">
					<view class="notice-type">{{ getNoticeTypeText(noticeData.noticeType) }}</view>
					<view class="notice-title">{{ noticeData.noticeTitle }}</view>
					<view class="notice-meta">
						<text class="meta-item">发布时间: {{ formatDateTime(noticeData.createTime) }}</text>
						<text class="meta-item" v-if="noticeData.startTime && noticeData.endTime">
							有效期: {{ formatDateRange(noticeData.startTime, noticeData.endTime) }}
						</text>
					</view>
				</view>

				<!-- 富文本内容区域 -->
				<view class="rich-text-container" v-if="!loading && noticeData && noticeData.noticeContent">
					<rich-text class="rich-text-content" :nodes="noticeData.noticeContent"></rich-text>
				</view>
				<view class="empty-container" v-else-if="!loading && (!noticeData || !noticeData.noticeContent)">
					<view class="empty-text">暂无公告内容</view>
				</view>
				<view class="loading-container" v-else>
					<view class="loading-text">加载中...</view>
				</view>
			</view>
	</view>
	</BaseContainer>
</template>

<script>
	import BaseContainer from '@/components/BaseWrapper.vue'

	export default {
		name: 'NoticeDetail',
		components: {
			BaseContainer
		},
		data() {
			return {
				noticeData: null,
				loading: true
			}
		},

		onLoad() {
			this.loadNoticeData()
		},

		onShow() {
			// 页面显示时重新加载数据，确保数据是最新的
			this.loadNoticeData()
		},

		methods: {
			// 加载公告数据
			loadNoticeData() {
				this.loading = true
				
				// 优先从全局变量获取数据
				let data = getApp().globalData.currentNotice
				
				// 如果全局变量没有，尝试从本地存储获取
				if (!data) {
					try {
						const storedData = uni.getStorageSync('currentNotice')
						if (storedData) {
							data = JSON.parse(storedData)
						}
					} catch (e) {
						console.error('Failed to parse notice from storage', e)
					}
				}

				if (data) {
					this.noticeData = {
						...data,
						// 处理富文本内容
						noticeContent: this.formatRichText(data.noticeContent || '')
					}
					this.loading = false
				} else {
					// 如果没有数据，显示错误信息
					console.warn('No notice data found in globalData or localStorage.')
					this.loading = false
				}
			},

			// 格式化富文本内容
			formatRichText(html) {
				if (!html) return ''
				
				// 处理图片自适应
				html = html.replace(
					/<img/gi,
					'<img style="max-width:100%;height:auto;display:block;"'
				)
				
				return html
			},

			// 获取公告类型文本
			getNoticeTypeText(type) {
				const typeMap = {
					'1': '公告',
					'2': '通知', 
					'3': '促销消息'
				}
				return typeMap[type] || '公告'
			},

			// 格式化日期时间
			formatDateTime(timestamp) {
				if (!timestamp) return ''
				const date = new Date(timestamp)
				return `${date.getFullYear()}-${(date.getMonth() + 1).toString().padStart(2, '0')}-${date.getDate().toString().padStart(2, '0')} ${date.getHours().toString().padStart(2, '0')}:${date.getMinutes().toString().padStart(2, '0')}`
			},

			// 格式化日期范围
			formatDateRange(start, end) {
				const startDate = new Date(start)
				const endDate = new Date(end)
				const startStr = `${startDate.getFullYear()}-${(startDate.getMonth() + 1).toString().padStart(2, '0')}-${startDate.getDate().toString().padStart(2, '0')}`
				const endStr = `${endDate.getFullYear()}-${(endDate.getMonth() + 1).toString().padStart(2, '0')}-${endDate.getDate().toString().padStart(2, '0')}`
				return `${startStr} 至 ${endStr}`
			},

			// 返回上一页
			goBack() {
				// 清除全局数据
				getApp().globalData.currentNotice = null
				uni.removeStorageSync('currentNotice')
				
				// 返回上一页
				uni.navigateBack()
			}
		}
	}
</script>

<style lang="scss" scoped>
	.notice-detail-container {
		min-height: 100vh;
		display: flex;
		flex-direction: column;
	}

	/* 内容区域 */
	.content-area {
		flex: 1;
		padding: 24rpx;
		background: #fff;
	}

	/* 公告头部信息 */
	.notice-header {
		margin-bottom: 32rpx;
		padding-bottom: 24rpx;
		border-bottom: 1rpx solid #f0f0f0;
	}

	.notice-type {
		display: inline-block;
		padding: 6rpx 16rpx;
		background: #ff6b00;
		color: #fff;
		font-size: 22rpx;
		border-radius: 16rpx;
		margin-bottom: 16rpx;
	}

	.notice-title {
		font-size: 32rpx;
		font-weight: bold;
		color: #333;
		line-height: 1.4;
		margin-bottom: 16rpx;
	}

	.notice-meta {
		display: flex;
		flex-direction: column;
		gap: 8rpx;
	}

	.meta-item {
		font-size: 24rpx;
		color: #999;
	}

	/* 富文本内容区域 */
	.rich-text-container {
		min-height: 400rpx;
		background: #fff;
		border-radius: 12rpx;
		overflow: hidden;
		word-wrap: break-word;
		word-break: break-all;

		.rich-text-content {
			width: 100%;
			line-height: 1.6;

			/* 富文本内图片自适应 */
			:deep(img) {
				max-width: 100% !important;
				height: auto !important;
				display: block;
				margin: 16rpx auto;
				border-radius: 8rpx;
				box-shadow: 0 2rpx 8rpx rgba(0, 0, 0, 0.1);
			}

			/* 富文本内段落样式优化 */
			:deep(p) {
				margin: 16rpx 0;
				line-height: 1.8;
				color: #333;
				font-size: 28rpx;
			}

			/* 富文本内标题样式 */
			:deep(h1),
			:deep(h2),
			:deep(h3),
			:deep(h4),
			:deep(h5),
			:deep(h6) {
				margin: 24rpx 0 16rpx 0;
				color: #333;
				font-weight: 600;
				line-height: 1.4;
			}

			:deep(h1) { font-size: 36rpx; }
			:deep(h2) { font-size: 34rpx; }
			:deep(h3) { font-size: 32rpx; }
			:deep(h4) { font-size: 30rpx; }
			:deep(h5) { font-size: 28rpx; }
			:deep(h6) { font-size: 26rpx; }

			/* 富文本内列表样式 */
			:deep(ul),
			:deep(ol) {
				margin: 16rpx 0;
				padding-left: 32rpx;
			}

			:deep(li) {
				margin: 8rpx 0;
				line-height: 1.6;
				color: #333;
				font-size: 28rpx;
			}

			/* 富文本内表格样式 */
			:deep(table) {
				width: 100%;
				border-collapse: collapse;
				margin: 16rpx 0;
				border: 1rpx solid #e0e0e0;
				border-radius: 8rpx;
				overflow: hidden;
			}

			:deep(th),
			:deep(td) {
				padding: 12rpx 16rpx;
				border: 1rpx solid #e0e0e0;
				text-align: left;
				font-size: 26rpx;
			}

			:deep(th) {
				background: #f5f5f5;
				font-weight: 600;
				color: #333;
			}

			:deep(td) {
				color: #666;
			}

			/* 富文本内链接样式 */
			:deep(a) {
				color: #009966;
				text-decoration: none;
			}

			/* 富文本内代码样式 */
			:deep(code) {
				background: #f5f5f5;
				padding: 4rpx 8rpx;
				border-radius: 4rpx;
				font-family: 'Courier New', monospace;
				font-size: 24rpx;
				color: #e91e63;
			}

			:deep(pre) {
				background: #f5f5f5;
				padding: 16rpx;
				border-radius: 8rpx;
				overflow-x: auto;
				margin: 16rpx 0;

				code {
					background: none;
					padding: 0;
					color: #333;
				}
			}

			/* 富文本内引用样式 */
			:deep(blockquote) {
				margin: 16rpx 0;
				padding: 16rpx 20rpx;
				background: #f9f9f9;
				border-left: 6rpx solid #009966;
				border-radius: 0 8rpx 8rpx 0;

				p {
					margin: 0;
					color: #666;
					font-style: italic;
				}
			}

			/* 富文本内分割线样式 */
			:deep(hr) {
				border: none;
				height: 2rpx;
				background: linear-gradient(90deg, transparent, #e0e0e0, transparent);
				margin: 32rpx 0;
			}
		}
	}

	/* 加载状态 */
	.loading-container {
		display: flex;
		align-items: center;
		justify-content: center;
		min-height: 400rpx;

		.loading-text {
			font-size: 28rpx;
			color: #999;
		}
	}

	/* 空状态 */
	.empty-container {
		display: flex;
		align-items: center;
		justify-content: center;
		min-height: 400rpx;

		.empty-text {
			font-size: 28rpx;
			color: #999;
		}
	}
</style>
