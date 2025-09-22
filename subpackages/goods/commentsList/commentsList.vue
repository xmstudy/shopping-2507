<template>
	<view class="comments-page">
		<!-- 商品信息卡片 -->
		<view class="product-card" v-if="goodsName">
			<text class="product-name">{{ goodsName }}</text>
			<text class="comment-count">共{{ total }}条评价</text>
		</view>

		<!-- 评论列表 -->
		<scroll-view 
			class="comments-scroll" 
			scroll-y 
			:refresher-enabled="true" 
			:refresher-triggered="refreshing"
			@refresherrefresh="onRefresh"
			@scrolltolower="onLoadMore"
			:scroll-top="scrollTop">
			
			<!-- 评论列表内容 -->
			<view class="comments-list" v-if="commentsList.length > 0">
				<view class="comment-item" v-for="(comment, index) in commentsList" :key="comment.id || index">
					<!-- 用户信息 -->
					<view class="comment-header">
						<image class="user-avatar" :src="comment.userAvatar" mode="aspectFill"></image>
						<view class="user-info">
							<text class="user-name">{{ comment.userName }}</text>
							<view class="rating-stars">
								<u-icon v-for="star in 5" :key="star" name="star-fill"
									:color="star <= comment.rating ? '#ffa500' : '#e0e0e0'" size="12"></u-icon>
							</view>
							<!-- 商品规格 -->
							<text class="comment-specs" v-if="comment.fullSpecs">{{ comment.fullSpecs }}</text>
						</view>
						<text class="comment-time">{{ formatTime(comment.createTime) }}</text>
					</view>

					<!-- 评论内容 -->
					<view class="comment-content">
						<text>{{ comment.content }}</text>
					</view>

					<!-- 评论图片 -->
					<view class="comment-images" v-if="comment.images && comment.images.length > 0 && comment.images.some(img => img && img.trim())">
						<image v-for="(img, imgIndex) in comment.images.filter(img => img && img.trim())" :key="imgIndex" 
							class="comment-image" :src="img" mode="aspectFill" 
							@click="previewImage(comment.images, imgIndex)">
						</image>
					</view>

					<!-- 商家回复 -->
					<view class="merchant-reply" v-if="comment.merchantReply">
						<view class="reply-header">
							<text class="reply-label">商家回复：</text>
							<text class="reply-time" v-if="comment.merchantReplyTime">{{ formatTime(comment.merchantReplyTime) }}</text>
						</view>
						<text class="reply-content">{{ comment.merchantReply }}</text>
					</view>
				</view>
			</view>

			<!-- 空状态 -->
			<view class="empty-state" v-else-if="!loading">
				<text>暂无评论</text>
			</view>

			<!-- 加载更多 -->
			<view class="load-more" v-if="commentsList.length > 0">
				<text v-if="loading">{{ loadingText }}</text>
				<text v-else-if="hasMore">上拉加载更多</text>
				<text v-else>没有更多了</text>
			</view>
		</scroll-view>
	</view>
</template>

<script>
	import { fetchGoodsCommentList } from '@/api/goods'

	export default {
		data() {
			return {
				// 页面参数
				goodsId: '',
				goodsName: '',
				
				// 页面状态
				loading: false,
				refreshing: false,
				loadingText: '加载中...',
				scrollTop: 0,
				
				// 评论数据
				commentsList: [],
				total: 0,
				pageNum: 1,
				pageSize: 10,
				hasMore: true
			}
		},

		onLoad(options) {
			// 获取页面参数
			this.goodsId = options.goodsId || ''
			this.goodsName = decodeURIComponent(options.goodsName || '')
			
			// 初始化页面
			this.initPage()
		},

		methods: {
			// 初始化页面
			initPage() {
				// 加载评论数据
				this.loadComments(true)
			},

			// 加载评论数据
			async loadComments(reset = false) {
				if (this.loading) return
				if (!reset && !this.hasMore) return

				try {
					this.loading = true
					this.loadingText = reset ? '加载中...' : '加载更多...'

					// 重置页码
					if (reset) {
						this.pageNum = 1
						this.hasMore = true
					}

					const params = {
						goodsId: this.goodsId,
						pageNum: this.pageNum,
						pageSize: this.pageSize
					}

					console.log('请求评论参数:', params)
					const res = await fetchGoodsCommentList(params)
					console.log('评论接口返回:', res)

					if (res.code === 200) {
						// 根据你之前修改的接口结构适配
						const rawComments = res.rows || []
						this.total = res.total || 0

						// 格式化评论数据
						const formattedComments = rawComments.map(comment => ({
							id: comment.id,
							goodsId: comment.goodsId,
							goodsName: comment.goodsName,
							goodsImage: comment.goodsImage,
							skuId: comment.skuId,
							fullSpecs: comment.fullSpecs,
							userName: comment.isAnonymous === 1 ? '匿名用户' : (comment.nickname || '用户'),
							userAvatar: comment.isAnonymous === 1 ? '/static/images/default-avatar.png' : (comment.avatarUrl || '/static/images/default-avatar.png'),
							rating: comment.score || 5,
							content: comment.content || '',
							images: this.parseCommentImages(comment.picUrls),
							videoUrl: comment.videoUrl,
							isAnonymous: comment.isAnonymous === 1,
							merchantReply: comment.merchantReply,
							merchantReplyTime: comment.merchantReplyTime,
							commentType: comment.commentType,
							createTime: comment.createTime,
							memberId: comment.memberId
						}))

						if (reset) {
							this.commentsList = formattedComments
						} else {
							this.commentsList.push(...formattedComments)
						}

						// 判断是否还有更多数据
						this.hasMore = formattedComments.length === this.pageSize
						
						if (this.hasMore) {
							this.pageNum++
						}

					} else {
						uni.showToast({
							title: res.msg || '获取评论失败',
							icon: 'none'
						})
					}

				} catch (error) {
					console.error('加载评论失败:', error)
					uni.showToast({
						title: '网络错误，请重试',
						icon: 'none'
					})
				} finally {
					this.loading = false
					this.refreshing = false
				}
			},

			// 下拉刷新
			onRefresh() {
				this.refreshing = true
				this.loadComments(true)
			},

			// 上拉加载更多
			onLoadMore() {
				if (!this.loading && this.hasMore) {
					this.loadComments(false)
				}
			},

			// 解析评论图片
			parseCommentImages(picUrls) {
				if (!picUrls) return []
				
				try {
					if (typeof picUrls === 'string') {
						// 先尝试JSON解析
						try {
							const parsed = JSON.parse(picUrls)
							if (Array.isArray(parsed)) {
								return parsed.filter(img => img && img.trim())
							}
						} catch (e) {
							// JSON解析失败，按逗号分隔处理
						}
						
						// 按逗号分隔并过滤空值
						return picUrls.split(',').map(img => img.trim()).filter(img => img)
					}
					
					if (Array.isArray(picUrls)) {
						return picUrls.filter(img => img && img.trim())
					}
					
					return []
				} catch (error) {
					console.error('解析评论图片失败:', error)
					return []
				}
			},

			// 预览图片
			previewImage(images, index) {
				const validImages = images.filter(img => img && img.trim())
				uni.previewImage({
					current: index,
					urls: validImages
				})
			},

			// 格式化时间
			formatTime(timeStr) {
				if (!timeStr) return ''
				try {
					const date = new Date(timeStr)
					const year = date.getFullYear()
					const month = String(date.getMonth() + 1).padStart(2, '0')
					const day = String(date.getDate()).padStart(2, '0')
					const hour = String(date.getHours()).padStart(2, '0')
					const minute = String(date.getMinutes()).padStart(2, '0')
					return `${year}-${month}-${day} ${hour}:${minute}`
				} catch (error) {
					return timeStr
				}
			}
		}
	}
</script>

<style lang="scss" scoped>
	.comments-page {
		height: 100vh;
		background: #f5f5f5;
		display: flex;
		flex-direction: column;
	}

	/* 自定义导航栏 */
	.custom-navbar {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		background: #fff;
		z-index: 999;
		box-shadow: 0 2rpx 8rpx rgba(0, 0, 0, 0.06);
	}

	.navbar-content {
		height: 44px;
		display: flex;
		align-items: center;
		justify-content: space-between;
		padding: 0 30rpx;
	}

	.navbar-left {
		width: 60rpx;
		height: 60rpx;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.navbar-title {
		font-size: 32rpx;
		font-weight: 600;
		color: #333;
	}

	.navbar-right {
		width: 60rpx;
	}

	/* 商品信息卡片 */
	.product-card {
		background: #fff;
		margin: 20rpx;
		padding: 30rpx;
		border-radius: 12rpx;
		box-shadow: 0 2rpx 8rpx rgba(0, 0, 0, 0.06);
	}

	.product-name {
		font-size: 28rpx;
		font-weight: 600;
		color: #333;
		display: block;
		margin-bottom: 12rpx;
	}

	.comment-count {
		font-size: 24rpx;
		color: #666;
	}

	/* 滚动容器 */
	.comments-scroll {
		flex: 1;
		height: 0;
	}

	/* 评论列表 */
	.comments-list {
		padding: 0 20rpx 20rpx;
	}

	.comment-item {
		background: #fff;
		margin-bottom: 20rpx;
		padding: 30rpx;
		border-radius: 12rpx;
		box-shadow: 0 2rpx 8rpx rgba(0, 0, 0, 0.06);
	}

	/* 评论头部 */
	.comment-header {
		display: flex;
		align-items: center;
		margin-bottom: 20rpx;
	}

	.user-avatar {
		width: 60rpx;
		height: 60rpx;
		border-radius: 50%;
		margin-right: 20rpx;
	}

	.user-info {
		flex: 1;
	}

	.user-name {
		font-size: 28rpx;
		font-weight: 600;
		color: #333;
		display: block;
		margin-bottom: 8rpx;
	}

	.rating-stars {
		display: flex;
		gap: 4rpx;
		margin-bottom: 8rpx;
	}

	.comment-specs {
		font-size: 22rpx;
		color: #999;
		display: block;
	}

	.comment-time {
		font-size: 24rpx;
		color: #999;
	}

	/* 评论内容 */
	.comment-content {
		margin-bottom: 20rpx;
		
		text {
			font-size: 28rpx;
			color: #333;
			line-height: 1.5;
		}
	}

	/* 评论图片 */
	.comment-images {
		display: flex;
		gap: 16rpx;
		flex-wrap: wrap;
		margin-bottom: 20rpx;
	}

	.comment-image {
		width: 120rpx;
		height: 120rpx;
		border-radius: 8rpx;
	}

	/* 商家回复 */
	.merchant-reply {
		margin-top: 20rpx;
		padding: 20rpx;
		background: #f8f9fa;
		border-radius: 12rpx;
		border-left: 4rpx solid #009966;
	}

	.reply-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 12rpx;
	}

	.reply-label {
		font-size: 24rpx;
		color: #009966;
		font-weight: 600;
	}

	.reply-time {
		font-size: 22rpx;
		color: #999;
	}

	.reply-content {
		font-size: 26rpx;
		color: #333;
		line-height: 1.5;
	}

	/* 空状态 */
	.empty-state {
		text-align: center;
		padding: 100rpx 0;
		
		text {
			font-size: 28rpx;
			color: #999;
		}
	}

	/* 加载更多 */
	.load-more {
		text-align: center;
		padding: 30rpx 0;
		
		text {
			font-size: 24rpx;
			color: #999;
		}
	}
</style>
