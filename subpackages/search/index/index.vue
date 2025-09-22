<template>
	<BaseContainer>
		<view class="search-container">
			<!-- 自定义头部 -->
			<view class="custom-header">
				<view class="back-btn" @click="goBack">
					<u-icon name="arrow-left" color="#333" size="20"></u-icon>
				</view>
				<view class="header-title">
					<text>搜索</text>
				</view>
				<view class="header-placeholder"></view>
			</view>
			
			<!-- 头部搜索区域 -->
			<view class="search-header">
			<view class="search-input-wrapper">
				<view class="search-icon">
					<u-icon name="search" color="#999" size="20"></u-icon>
				</view>
				<input 
					class="search-input" 
					v-model="searchKeyword" 
					placeholder="搜索商品" 
					@input="onSearchInput"
					@confirm="onSearchConfirm"
					focus
				/>
				<view class="clear-icon" v-if="searchKeyword" @click="clearSearch">
					<u-icon name="close" color="#999" size="16"></u-icon>
				</view>
				<view class="search-btn" @click="onSearchConfirm" v-if="searchKeyword">
					<text>搜索</text>
				</view>
			</view>
		</view>

		<!-- 搜索历史记录 -->
		<view class="search-history" v-if="searchHistory.length > 0 && !searchKeyword">
			<view class="section-header">
				<text class="section-title">搜索历史</text>
				<view class="clear-history" @click="clearHistory">
					<u-icon name="trash" color="#999" size="16"></u-icon>
				</view>
			</view>
			<view class="history-list">
				<view class="history-item" v-for="(item, index) in searchHistory" :key="index" @click="searchByHistory(item)">
					<text class="history-text">{{ item }}</text>
					<view class="delete-history" @click.stop="deleteHistoryItem(index)">
						<u-icon name="close" color="#999" size="14"></u-icon>
					</view>
				</view>
			</view>
		</view>

		<!-- 搜索结果 -->
		<view class="search-results" v-if="searchResults.length > 0 || loading">
			<view class="section-header">
				<text class="section-title">搜索结果</text>
				<text class="result-count">共{{ searchResults.length }}条</text>
			</view>
			<view class="result-list">
				<!-- 加载状态 -->
				<view class="loading-container" v-if="loading">
					<u-loading-icon mode="spinner" size="40"></u-loading-icon>
					<text class="loading-text">搜索中...</text>
				</view>
				
				<!-- 搜索结果 -->
				<view class="result-item" v-for="(item, index) in searchResults" :key="index" @click="goToProductDetail(item)">
					<image class="result-img" :src="item.img" mode="aspectFill"></image>
					<view class="result-info">
						<view class="result-header">
							<view class="result-name">{{ item.name }}</view>
							<view class="result-price-section">
								<text class="result-price">¥{{ item.price }}</text>
								<text class="result-original-price" v-if="item.originalPrice && item.originalPrice > item.price">¥{{ item.originalPrice }}</text>
							</view>
						</view>
						<view class="result-desc">{{ item.desc }}</view>
						<view class="result-stats">
							<text class="stat-item">销量: {{ item.soldNum || 0 }}</text>
						</view>
					</view>
				</view>
			</view>
		</view>

		<!-- 热门搜索推荐 -->
		<view class="hot-search" v-if="!searchKeyword && searchResults.length === 0">
			<view class="section-header">
				<text class="section-title">热门搜索</text>
			</view>
			<view class="hot-list">
				<view class="hot-item" v-for="(item, index) in hotSearch" :key="index" @click="searchByHot(item)">
					<view class="hot-rank rank-first" v-if="index === 0">
						1
					</view>
					<view class="hot-rank rank-second" v-else-if="index === 1">
						2
					</view>
					<view class="hot-rank rank-third" v-else-if="index === 2">
						3
					</view>
					<view class="hot-rank rank-default" v-else>
						{{ index + 1 }}
					</view>
					<text class="hot-text">{{ item }}</text>
				</view>
			</view>
		</view>
		</view>
	</BaseContainer>
</template>

<script>
	import { fetchGoodsList } from '@/api/goods'
	import BaseContainer from '@/components/BaseWrapper.vue'
	
	export default {
		components: {
			BaseContainer
		},
					data() {
				return {
					searchKeyword: '', // 搜索关键词
					searchHistory: [], // 搜索历史（从本地存储加载）
					hotSearch: [], // 热门搜索（从推荐商品获取）
					searchResults: [], // 搜索结果
					loading: false, // 加载状态
					searchTimer: null // 搜索防抖定时器
				}
			},
		mounted() {
			this.loadHotSearch()
			this.loadSearchHistory()
		},
		beforeDestroy() {
			// 清理定时器
			if (this.searchTimer) {
				clearTimeout(this.searchTimer)
			}
		},
		methods: {
			// 加载搜索历史记录
			loadSearchHistory() {
				try {
					const history = uni.getStorageSync('searchHistory')
					if (history && Array.isArray(history)) {
						this.searchHistory = history
					}
				} catch (error) {
					console.error('加载搜索历史失败:', error)
					this.searchHistory = []
				}
			},
			
			// 保存搜索历史记录到本地
			saveSearchHistory() {
				try {
					uni.setStorageSync('searchHistory', this.searchHistory)
				} catch (error) {
					console.error('保存搜索历史失败:', error)
				}
			},
			
			// 加载热门搜索数据
			async loadHotSearch() {
				try {
					const params = {
						pageSize: 10,
						pageNo: 1,
						isRecommend: 1
					}
					
					const res = await fetchGoodsList(params)
					console.log('热门搜索数据:', res)
					
					if (res.code == 200 && res.rows) {
						// 从推荐商品中提取商品名称作为热门搜索
						this.hotSearch = res.rows.map(item => item.goodsName)
					}
				} catch (error) {
					console.error('获取热门搜索失败:', error)
					// 如果获取失败，使用默认数据
					this.hotSearch = ['新鲜蔬菜', '有机水果', '高山茶叶', '时令水果', '绿色食品', '健康食材']
				}
			},
			
			// 搜索输入
			onSearchInput() {
				// 如果输入为空，立即清空搜索结果
				if (!this.searchKeyword.trim()) {
					this.searchResults = []
					this.loading = false
				}
				// 输入时不再自动触发搜索，只有确认时才搜索
			},
			
			// 确认搜索
			onSearchConfirm() {
				if (this.searchKeyword.trim()) {
					this.performSearch() // 在调用接口时就保存到历史记录
				}
			},
			
			// 清除搜索
			clearSearch() {
				this.searchKeyword = ''
				this.searchResults = []
			},
			
			// 添加到历史记录
			addToHistory(keyword) {
				// 如果已存在，先移除旧的
				const existingIndex = this.searchHistory.indexOf(keyword)
				if (existingIndex !== -1) {
					this.searchHistory.splice(existingIndex, 1)
				}
				
				// 添加到最前面
				this.searchHistory.unshift(keyword)
				
				// 限制历史记录数量
				if (this.searchHistory.length > 10) {
					this.searchHistory = this.searchHistory.slice(0, 10)
				}
				
				// 保存到本地存储
				this.saveSearchHistory()
			},
			
			// 执行搜索
			async performSearch(addToHistory = false) {
				if (!this.searchKeyword.trim()) {
					this.searchResults = []
					return
				}
				
				// 在调用接口前就保存到搜索历史
				this.addToHistory(this.searchKeyword.trim())
				
				this.loading = true
				try {
					const params = {
						pageSize: 20,
						pageNo: 1,
						goodsName: this.searchKeyword.trim()
					}
					
					const res = await fetchGoodsList(params)
					console.log('搜索结果:', res)
					
					if (res.code == 200 && res.rows) {
						// 将API返回的商品数据转换为页面需要的格式
						this.searchResults = res.rows.map(item => ({
							id: item.goodsId,
							img: item.goodsImage,
							name: item.goodsName,
							price: item.promotionPrice || item.price,
							originalPrice: item.price,
							soldNum: item.soldNum || 0,
							desc: item.sellingPoint || '优质商品，值得信赖'
						}));
					} else {
						this.searchResults = []
					}
				} catch (error) {
					console.error('搜索失败:', error)
					uni.showToast({
						title: '搜索失败，请重试',
						icon: 'none'
					})
					this.searchResults = []
				} finally {
					this.loading = false
				}
			},
			
			// 点击历史记录搜索
			searchByHistory(keyword) {
				this.searchKeyword = keyword
				// 点击历史记录时直接搜索（会自动保存到历史记录顶部）
				this.performSearch()
			},
			
			// 删除单个历史记录
			deleteHistoryItem(index) {
				this.searchHistory.splice(index, 1)
				// 保存到本地存储
				this.saveSearchHistory()
			},
			
			// 清空所有历史记录
			clearHistory() {
				uni.showModal({
					title: '确认清空',
					content: '确定要清空所有搜索历史吗？',
					success: (res) => {
						if (res.confirm) {
							this.searchHistory = []
							// 保存到本地存储
							this.saveSearchHistory()
							uni.showToast({
								title: '已清空历史',
								icon: 'success'
							})
						}
					}
				})
			},
			
			// 跳转到商品详情
			goToProductDetail(item) {
				uni.navigateTo({
					url: `/subpackages/goods/detail/detail?goodsId=${item.id}`
				})
			},
			
			// 点击热门搜索
			searchByHot(keyword) {
				this.searchKeyword = keyword
				// 点击热门搜索时直接搜索（会自动保存到历史记录）
				this.performSearch()
			},
			
			// 回退处理
			goBack() {
				const pages = getCurrentPages()
				if (pages.length > 1) {
					// 有上一级页面，执行返回
					uni.navigateBack({
						delta: 1
					})
				} else {
					// 没有上一级页面，重新启动到首页，不留历史记录
					uni.reLaunch({
						url: '/pages/index/index'
					})
				}
			}
		}
	}
</script>

<style lang="scss" scoped>
	.search-container {
		min-height: 100vh;
		background: #f8f9fa;
		padding: 0;
	}

	/* 自定义头部 */
	.custom-header {
		display: flex;
		align-items: center;
		justify-content: space-between;
		padding: 20rpx 24rpx;
		background: #fff;
		border-bottom: 1rpx solid #f0f0f0;
		position: sticky;
		top: 0;
		z-index: 100;

		.back-btn {
			width: 60rpx;
			height: 60rpx;
			display: flex;
			align-items: center;
			justify-content: center;
			border-radius: 50%;
			background: #f5f5f5;
			transition: all 0.2s ease;

			&:active {
				background: #e0e0e0;
				transform: scale(0.95);
			}
		}

		.header-title {
			flex: 1;
			display: flex;
			align-items: center;
			justify-content: center;

			text {
				font-size: 36rpx;
				font-weight: bold;
				color: #333;
			}
		}

		.header-placeholder {
			width: 60rpx;
			height: 60rpx;
		}
	}

	/* 头部搜索区域 */
	.search-header {
		background: #fff;
		padding: 20rpx 24rpx 32rpx;
		position: relative;
		border-bottom: 1rpx solid #f0f0f0;

		.search-input-wrapper {
			display: flex;
			align-items: center;
			background: #fff;
			border-radius: 24rpx;
			padding: 16rpx 20rpx;
			box-shadow: 0 4rpx 16rpx rgba(0, 0, 0, 0.1);
			position: relative;

			.search-icon {
				margin-right: 16rpx;
				flex-shrink: 0;
			}

			.search-input {
				flex: 1;
				font-size: 28rpx;
				color: #333;
				background: transparent;
				border: none;
				outline: none;
			}

			.clear-icon {
				margin-left: 16rpx;
				padding: 8rpx;
				border-radius: 50%;
				background: #f5f5f5;
				transition: all 0.2s ease;
				flex-shrink: 0;

				&:active {
					background: #e0e0e0;
					transform: scale(0.9);
				}
			}

			.search-btn {
				margin-left: 16rpx;
				padding: 12rpx 20rpx;
				background: linear-gradient(135deg, #a6e22e 0%, #88cc00 100%);
				border-radius: 20rpx;
				transition: all 0.2s ease;
				flex-shrink: 0;

				text {
					font-size: 26rpx;
					color: #fff;
					font-weight: bold;
				}

				&:active {
					background: linear-gradient(135deg, #88cc00 0%, #6a9900 100%);
					transform: scale(0.95);
				}
			}
		}
	}

	/* 搜索历史记录 */
	.search-history {
		background: #fff;
		padding: 24rpx;
		margin-bottom: 16rpx;

		.section-header {
			display: flex;
			align-items: center;
			justify-content: space-between;
			margin-bottom: 20rpx;

			.section-title {
				font-size: 30rpx;
				font-weight: bold;
				color: #333;
			}

			.clear-history {
				padding: 8rpx;
				border-radius: 50%;
				transition: all 0.2s ease;

				&:active {
					background: #f5f5f5;
					transform: scale(0.9);
				}
			}
		}

		.history-list {
			display: flex;
			flex-wrap: wrap;
			gap: 16rpx;

			.history-item {
				display: flex;
				align-items: center;
				background: #f8f9fa;
				border-radius: 20rpx;
				padding: 12rpx 16rpx;
				border: 1rpx solid #e9ecef;
				transition: all 0.2s ease;
				max-width: 100%;

				&:active {
					background: #e9ecef;
					transform: scale(0.95);
				}

				.history-text {
					font-size: 26rpx;
					color: #666;
					white-space: nowrap;
					overflow: hidden;
					text-overflow: ellipsis;
					max-width: 200rpx;
				}

				.delete-history {
					margin-left: 12rpx;
					padding: 4rpx;
					border-radius: 50%;
					transition: all 0.2s ease;

					&:active {
						background: #e0e0e0;
						transform: scale(0.8);
					}
				}
			}
		}
	}

	/* 搜索结果 */
	.search-results {
		background: #fff;
		padding: 24rpx;

		.section-header {
			display: flex;
			align-items: center;
			justify-content: space-between;
			margin-bottom: 24rpx;

			.section-title {
				font-size: 30rpx;
				font-weight: bold;
				color: #333;
			}

			.result-count {
				font-size: 24rpx;
				color: #999;
			}
		}

		.result-list {
			display: flex;
			flex-direction: column;
			gap: 20rpx;
		}

		.loading-container {
			display: flex;
			flex-direction: column;
			align-items: center;
			justify-content: center;
			padding: 60rpx 0;
			gap: 20rpx;

			.loading-text {
				font-size: 26rpx;
				color: #666;
			}
		}

		.result-item {
			background: linear-gradient(135deg, #fff 0%, #f8fff0 100%);
			border-radius: 20rpx;
			overflow: hidden;
			box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.08);
			display: flex;
			flex-direction: row;
			padding: 20rpx;
			gap: 16rpx;
			border: 1rpx solid rgba(166, 226, 46, 0.1);
			transition: all 0.3s ease;

			&:active {
				transform: translateY(-2rpx);
				box-shadow: 0 6rpx 16rpx rgba(0, 0, 0, 0.12);
			}

			.result-img {
				width: 120rpx;
				height: 120rpx;
				border-radius: 16rpx;
				object-fit: cover;
				background: linear-gradient(135deg, #f0f8e0 0%, #e8f5d0 100%);
				flex-shrink: 0;
				box-shadow: 0 2rpx 8rpx rgba(166, 226, 46, 0.1);
			}

			.result-info {
				flex: 1;
				display: flex;
				flex-direction: column;
				gap: 8rpx;
				min-width: 0;
			}

			.result-header {
				display: flex;
				align-items: center;
				justify-content: space-between;
				margin-bottom: 8rpx;
			}

			.result-name {
				font-size: 28rpx;
				font-weight: bold;
				color: #2e7d32;
				line-height: 1.2;
				flex: 1;
			}

			.result-price-section {
				display: flex;
				align-items: center;
				gap: 8rpx;
				margin-left: 16rpx;
			}

			.result-price {
				font-size: 26rpx;
				color: #e54d42;
				font-weight: bold;
			}

			.result-original-price {
				font-size: 22rpx;
				color: #999;
				text-decoration: line-through;
			}

			.result-desc {
				font-size: 24rpx;
				color: #666;
				line-height: 1.4;
				margin-bottom: 8rpx;
				overflow: hidden;
				text-overflow: ellipsis;
				display: -webkit-box;
				-webkit-line-clamp: 2;
				-webkit-box-orient: vertical;
			}

			.result-stats {
				display: flex;
				gap: 12rpx;
				font-size: 22rpx;
				color: #666;

				.stat-item {
					background: linear-gradient(135deg, #f0f8e0 0%, #e8f5d0 100%);
					padding: 6rpx 12rpx;
					border-radius: 12rpx;
					border: 1rpx solid rgba(166, 226, 46, 0.2);
					font-weight: 500;
				}
			}
		}
	}

	/* 热门搜索推荐 */
	.hot-search {
		background: #fff;
		padding: 24rpx;

		.section-header {
			display: flex;
			align-items: center;
			justify-content: space-between;
			margin-bottom: 24rpx;

			.section-title {
				font-size: 30rpx;
				font-weight: bold;
				color: #333;
			}
		}

		.hot-list {
			display: flex;
			flex-direction: column;
			gap: 16rpx;
		}

		.hot-item {
			display: flex;
			align-items: center;
			padding: 20rpx;
			background: #f8f9fa;
			border-radius: 16rpx;
			border: 1rpx solid #e9ecef;
			transition: all 0.3s ease;
			position: relative;

			&:active {
				background: #e9ecef;
				transform: scale(0.98);
			}

			.hot-rank {
				width: 40rpx;
				height: 40rpx;
				border-radius: 50%;
				display: flex;
				align-items: center;
				justify-content: center;
				font-size: 24rpx;
				font-weight: bold;
				margin-right: 20rpx;
				flex-shrink: 0;

				&.rank-first {
					background: linear-gradient(135deg, #ff4757 0%, #ff3742 100%);
					color: #fff;
					box-shadow: 0 4rpx 12rpx rgba(255, 71, 87, 0.3);
				}

				&.rank-second {
					background: linear-gradient(135deg, #ffa502 0%, #ff9500 100%);
					color: #fff;
					box-shadow: 0 4rpx 12rpx rgba(255, 165, 2, 0.3);
				}

				&.rank-third {
					background: linear-gradient(135deg, #2ed573 0%, #1e90ff 100%);
					color: #fff;
					box-shadow: 0 4rpx 12rpx rgba(46, 213, 115, 0.3);
				}

				&.rank-default {
					background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
					color: #666;
					border: 1rpx solid #dee2e6;
					box-shadow: 0 2rpx 6rpx rgba(0, 0, 0, 0.1);
				}
			}

			.hot-text {
				font-size: 28rpx;
				color: #333;
				font-weight: 500;
				flex: 1;
				overflow: hidden;
				text-overflow: ellipsis;
				display: -webkit-box;
				-webkit-line-clamp: 2;
				-webkit-box-orient: vertical;
				line-height: 1.4;
			}
		}
	}
</style>
