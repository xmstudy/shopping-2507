<template>
	<view class="page">
		<!-- 固定顶部自定义标题栏 -->
		<view class="nav-fixed" :style="navInlineStyle">
			<view class="nav-inner">
				<text class="nav-title">雷波农特</text>
			</view>
		</view>

		<!-- 页面主体：用 padding-top 把内容顶到自定义导航栏下方 -->
		<view class="page-body" :style="{ paddingTop: contentTopPx + 'px' }">

			<!-- 搜索条（第二部分） -->
			<view class="search-wrap" @click="toSearch">
				<view class="search-box">
					<!-- 如果你用 uView2，可换成 <u-icon name="search" size="18" /> -->
					<u-icon name="search" class="icon-search"></u-icon>
					<text class="search-text">搜索商品</text>
				</view>
			</view>

			<!-- 排序区域 -->
			<view class="sort-wrap">
				<view class="sort-item" @click="sortBy('default')" :class="{ active: currentSort === 'default' }">
					<text>综合</text>
				</view>
				<view class="sort-item" @click="sortBy('sales')" :class="{ active: currentSort === 'sales' }">
					<text>销量</text>
				</view>
				<view class="sort-item price-sort" @click="togglePriceSort"
					:class="{ active: currentSort === 'price' }">
					<text>价格</text>
					<view class="sort-arrows">
						<u-icon name="arrow-up" size="12"
							:color="currentSort === 'price' && sortOrder === 'asc' ? '#07c160' : '#ccc'"></u-icon>
						<u-icon name="arrow-down" size="12"
							:color="currentSort === 'price' && sortOrder === 'desc' ? '#07c160' : '#ccc'"></u-icon>
					</view>
				</view>
			</view>

			<!-- 商品区（第三部分：左右分栏） -->
			<view class="goods-wrap">
				<!-- 左侧：分类 Tab（可滚动） -->
				<scroll-view scroll-y class="left-tabs">
					<view v-for="(c, i) in categories" :key="c.id" class="tab-item"
						:class="{ active: i === activeIndex }" @tap="tapCat(i)">
						{{ c.name }}
					</view>
				</scroll-view>

				<!-- 右侧：商品列表（可滚动） -->
				<scroll-view scroll-y class="right-list" :enable-flex="true">
					<view class="goods-content">
						<!-- 顶部Banner：展示id为1的文章 -->
						<view class="banner-container" @click="goToBannerDetail" v-if="bannerArticle">
							<image class="banner-image" :src="bannerArticle.image" mode="aspectFill"></image>
							<view class="banner-overlay">
								<text class="banner-title">{{ bannerArticle.name }}</text>
							</view>
						</view>
						<!-- 默认占位Banner -->
						<view class="banner-placeholder" v-else></view>

						<!-- 加载状态 -->
						<view class="loading-container" v-if="goodsLoading">
							<view class="loading-text">加载中...</view>
						</view>
						<!-- 商品项 -->
						<view class="item" v-for="(g, k) in currentGoods" :key="g.goodsId || k"
							@click="goToGoodsDetail(currentGoods,k)">
							<view class="thumb">
								<image class="thumb-image" :src="g.img" mode="aspectFill" :lazy-load="true"
									@load="onImageLoad(k)" @error="onImageError(k)"></image>
								<view class="image-skeleton" v-if="!g.imageLoaded">
									<view class="skeleton-shimmer"></view>
								</view>
								<!-- 左上角热销图标 -->
								<view class="item-badge" v-if="g.isHot">
									<image class="badge-icon" src="/static/images/hot-icon.png" mode="aspectFit">
									</image>
								</view>
								<view class="item-badge" v-else-if="g.originalPrice && g.originalPrice > g.price">
									<image class="badge-icon" src="/static/images/saleicon.png" mode="aspectFit">
									</image>
								</view>
							</view>

							<view class="info">
								<view class="title">{{ g.title }}</view>
								<view class="tags">{{ g.tags }}</view>
								<view class="sales-info">
									<text class="sales-label">销量: </text>
									<text class="sales-count">{{ g.virtualSoldNum || 0 }}</text>
								</view>
								<view class="bottom">
									<view class="price-section">
										<text class="price">¥{{ g.price }}</text>
										<text class="original-price"
											v-if="g.originalPrice && g.originalPrice > g.price">¥{{ g.originalPrice }}</text>
									</view>
									<view class="btn-add" @tap.stop="()=>addToCart(g)">
										<image class="add-cart-icon" src="/static/images/add-cart.png" mode="aspectFit">
										</image>
									</view>
								</view>
							</view>
						</view>

						<!-- 空状态 -->
						<view class="empty-container" v-if="!goodsLoading && currentGoods.length === 0">
							<view class="empty-text">暂无商品</view>
						</view>
					</view>
				</scroll-view>
			</view>
		</view>
		<!-- SKU选择弹窗 -->
		<u-popup v-if="showSku" :show="showSku" @close="closeSkuSelector" mode="bottom" :round="20" :overlay="true"
			:closeOnClickOverlay="true" :safeAreaInsetBottom="true">
			<view class="sku-popup">
				<view class="sku-header">
					<image class="sku-product-image" :src="getSkuImage()" mode="aspectFill"></image>
					<view class="sku-product-info">
						<text class="sku-product-name">{{ productName }}</text>
						<view class="sku-product-price">
							<text class="sku-price-symbol">¥</text>
							<text class="sku-price-value">{{ selectedSkuPrice }}</text>
						</view>
					</view>
					<view class="sku-close" @click="closeSkuSelector">
						<u-icon name="close" color="#999" size="20"></u-icon>
					</view>
				</view>

				<view class="sku-options">
					<view class="sku-option-group" v-for="(spec, specIndex) in skuOptions.spec" :key="specIndex">
						<text class="option-title">{{ spec.name }}</text>
						<view class="option-items">
							<view class="option-item" v-for="(value, valueIndex) in spec.values" :key="valueIndex"
								:class="{ active: spec.selectedValue === value }"
								@click="selectSpecValue(specIndex, value)">
								<view class="option-content">
									<text>{{ value }}</text>
									<text class="sku-price"
										v-if="getSkuPrice(specIndex, value)">¥{{ getSkuPrice(specIndex, value) }}</text>
								</view>
							</view>
						</view>
					</view>

					<view class="quantity-selector">
						<text class="quantity-title">数量</text>
						<view class="quantity-control">
							<view class="quantity-btn" @click="decreaseQuantity">
								<u-icon name="minus" color="#666" size="16"></u-icon>
							</view>
							<text class="quantity-value">{{ quantity }}</text>
							<view class="quantity-btn" @click="increaseQuantity">
								<u-icon name="plus" color="#666" size="16"></u-icon>
							</view>
						</view>
					</view>
				</view>

				<view class="sku-footer">
					<view class="sku-total">
						<text>总计：</text>
						<text class="total-price">¥{{ totalPrice }}</text>
					</view>
					<view class="sku-action" @click="confirmSkuAction">
						<text>{{ skuActionType === 'cart' ? '加入购物车' : '确认购买' }}</text>
					</view>
				</view>
			</view>
		</u-popup>
		<IconComVue />
	</view>
</template>

<script>
	import {
		fetchGoodsTreeList,
		fetchGoodsList,
		fetchGoodsDetail
	} from '@/api/goods'

	import {
		AddCart
	} from '@/api/cart'
	import {
		fetchHomeBanner
	} from '@/api/index'
	import IconComVue from '../../components/IconCom.vue'
	export default {
		components: {
			IconComVue
		},
		data() {
			return {
				// 顶部安全/胶囊
				statusBarHeight: 0,
				capsuleHeight: 44, // 默认值，实际小程序里会用胶囊高度覆盖
				capsuleTop: 0, // 胶囊距离顶部的距离

				// 分类数据 - 从接口获取
				categories: [],
				activeIndex: 0,
				// 当前分类的商品列表
				currentGoodsList: [],
				originalGoodsList: [], // 存储原始商品列表，用于排序
				goodsLoading: false,
				// Banner文章数据
				bannerArticle: null,

				// 排序相关
				currentSort: 'default', // 当前排序方式：default, sales, price
				sortOrder: 'desc', // 排序顺序：asc升序, desc降序

				// SKU弹窗相关
				showSku: false,
				skuActionType: 'cart', // 'cart' 或 'buy'
				selectedSkuIndex: 0,
				quantity: 1,
				productName: '',
				currentPrice: 0,
				originalPrice: 0,
				skuOptions: {
					spec: []
				},
				productData: null,
				productImages: [],
				_skuSpecsCache: null // SKU缓存
			}
		},
		computed: {
			// 内联样式：顶部固定栏 paddingTop=胶囊top，height=胶囊height
			navInlineStyle() {
				return `padding-top:${this.capsuleTop}px;background:#fff;`
			},
			// 导航栏总高度：胶囊top + 胶囊height
			navTotalHeight() {
				return this.capsuleTop + this.capsuleHeight
			},
			// 内容区顶部偏移：导航栏总高度
			contentTopPx() {
				return this.navTotalHeight
			},
			currentGoods() {
				console.log(' this.currentGoodsList ', this.currentGoodsList);
				return this.currentGoodsList || []
			},

			// SKU相关计算属性
			selectedSkuPrice() {
				const selectedSku = this.getSelectedSku()
				if (selectedSku) {
					// 优先使用促销价格，其次使用原价
					return selectedSku.promotionPrice || selectedSku.price || this.currentPrice
				}
				return this.currentPrice || 0
			},

			totalPrice() {
				return (this.selectedSkuPrice * this.quantity).toFixed(2)
			}
		},
		mounted() {
			const sys = uni.getSystemInfoSync()
			this.statusBarHeight = sys.statusBarHeight || 0

			// #ifdef MP-WEIXIN
			try {
				const rect = wx.getMenuButtonBoundingClientRect()
				if (rect && rect.height) {
					this.capsuleHeight = rect.height
					this.capsuleTop = rect.top + 5
				}
			} catch (e) {}
			// #endif

			// 其他端给个合理默认
			// #ifndef MP-WEIXIN
			this.capsuleHeight = 44
			this.capsuleTop = this.statusBarHeight
			// #endif

			// 获取商品分类数据
			this.loadCategories()
			// 获取Banner文章数据
			this.loadBannerArticle()
		},
		methods: {
			toSearch() {
				uni.navigateTo({
					url: "/subpackages/search/index/index"
				})
			},
			tapCat(i) {
				this.activeIndex = i
				console.log('切换到分类:', this.categories[i].name)
				// 根据选中的二级分类加载商品
				this.loadGoodsByCategory(this.categories[i])
			},

			// 加载商品分类数据
			async loadCategories() {
				try {
					// TODO fetchGoodsTreeList 会在这里使用
					const res = await fetchGoodsTreeList()
					if (res.code == 200 && res.data && res.data.length > 0) {
						// 处理分类数据，以第二层为左侧分类
						this.categories = []
						res.data.forEach(firstLevel => {
							if (firstLevel.children && firstLevel.children.length > 0) {
								firstLevel.children.forEach(secondLevel => {
									this.categories.push({
										id: secondLevel.id,
										name: secondLevel.name,
										parentId: firstLevel.id
									})
								})
							}
						})

						// 如果有分类数据，设置初始选中并加载第一个分类的商品
						if (this.categories.length > 0) {
							this.activeIndex = 0
							this.loadGoodsByCategory(this.categories[0])
						}
					} else {
						// 如果接口没有数据，显示空分类
						console.log('分类接口无数据')
						this.categories = []
					}
				} catch (error) {
					console.error('获取分类数据失败:', error)
					// 异常时显示空分类
					this.categories = []
				}
			},

			// 根据分类加载商品
			async loadGoodsByCategory(category) {
				try {
					this.goodsLoading = true

					// 直接使用当前二级分类的ID获取商品
					// TODO fetchGoodsList 会在这里使用
					const params = {
						pageSize: 20,
						pageNo: 1,
						category2Id: category.id // 直接使用二级分类ID
					}
					const res = await fetchGoodsList(params)
					if (res.code == 200 && res.rows && res.rows.length > 0) {
						const mappedGoods = res.rows.map(item => ({
							goodsId: item.goodsId,
							title: item.goodsName,
							price: parseFloat(item.promotionPrice || item.price || 0),
							originalPrice: parseFloat(item.price || 0),
							tags: item.sellingPoint || '精选商品',
							img: item.image || item.goodsImage || item.img,
							isHot: item.isHot == 1,
							soldNum: parseInt(item.soldNum || 0),
							virtualSoldNum: parseInt(item.virtualSoldNum || 0),
							imageLoaded: false
						}));

						// 保存原始数据
						this.originalGoodsList = [...mappedGoods];
						// 应用当前排序
						this.applySorting(mappedGoods);
					} else {
						// 如果接口没有数据，显示空列表
						console.log('商品列表接口无数据')
						this.currentGoodsList = []
					}
				} catch (error) {
					console.error('加载商品失败:', error)
					// 异常时显示空列表
					this.currentGoodsList = []
				} finally {
					this.goodsLoading = false
				}
			},

			// 跳转商品详情
			goToGoodsDetail(list, idx) {
				let item = list[idx]
				console.log('跳转商品详情，商品信息:', item)
				console.log('商品所有字段:', Object.keys(item))

				// 尝试多个可能的字段名
				const goodsId = item.goodsId || item.id || item.goods_id || item.productId || item.product_id

				console.log('提取到的goodsId:', goodsId)

				if (!goodsId) {
					console.error('商品ID不存在，完整商品信息:', JSON.stringify(item, null, 2))
					uni.showToast({
						title: '商品信息异常',
						icon: 'none',
						duration: 3000
					})
					return
				}

				console.log('即将跳转到详情页，goodsId:', goodsId)
				uni.navigateTo({
					url: `/subpackages/goods/detail/detail?goodsId=${goodsId}`
				})
			},
			// 添加到购物车
			async addToCart(item) {
				try {
					if (!item.goodsId) {
						console.error('商品信息不存在')
						return
					}

					// 获取商品详情
					const res = await fetchGoodsDetail(item.goodsId)
					if (res.code === 200 && res.data) {
						const data = res.data
						console.log('获取到的商品详情数据:', data)

						// 处理商品数据
						this.processProductData(data)

						// 显示SKU选择弹窗让用户选择数量和规格
						this.skuActionType = 'cart'
						this.showSku = true
						console.log('显示SKU弹窗', {
							showSku: this.showSku,
							productName: this.productName,
							skuOptions: this.skuOptions
						})
					} else {
						uni.showToast({
							title: res.msg || '获取商品信息失败',
							icon: 'none',
							duration: 2000
						})
					}
				} catch (error) {
					console.error('添加购物车失败:', error)
					uni.showToast({
						title: '添加失败，请重试',
						icon: 'none',
						duration: 2000
					})
				}
			},

			// 图片加载处理
			onImageLoad(index) {
				if (this.currentGoodsList[index]) {
					this.$set(this.currentGoodsList[index], 'imageLoaded', true)
				}
			},

			// 图片错误处理
			onImageError(index) {
				if (this.currentGoodsList[index]) {
					this.$set(this.currentGoodsList[index], 'imageLoaded', true)
					// 设置默认图片
					this.$set(this.currentGoodsList[index], 'img',
						'https://pic1.imgdb.cn/item/6895daa458cb8da5c812a9a6.jpg')
				}
			},

			// 获取Banner数据
			async loadBannerArticle() {
				try {
					// TODO fetchHomeBanner 会在这里使用 - 参考 home 界面实现
					const res = await fetchHomeBanner({
						type: 2
					})
					if (res.code == 200 && res.data && res.data.length > 0) {
						// 取第一个数据作为Banner展示，参考home界面的数据映射方式
						const bannerItem = res.data[0]
						this.bannerArticle = {
							id: bannerItem.id || 1,
							name: bannerItem.title || bannerItem.name || '雷波农特',
							image: bannerItem.image || bannerItem.pic || bannerItem.url || bannerItem.imageUrl,
							content: bannerItem.content || bannerItem.desc || bannerItem.description || ''
						}
					} else {
						console.log('Banner接口无数据，使用默认数据')
						// 如果接口没有数据，使用默认banner
						this.bannerArticle = {
							id: 1,
							name: '雷波农特',
							image: 'https://pic1.imgdb.cn/item/6895db6e58cb8da5c812aa02.jpg',
							content: '雷波特色农产品展示'
						}
					}
				} catch (error) {
					console.error('获取Banner数据失败:', error)
					// 异常情况使用默认数据
					this.bannerArticle = {
						id: 1,
						name: '雷波农特',
						image: 'https://pic1.imgdb.cn/item/6895db6e58cb8da5c812aa02.jpg',
						content: '雷波特色农产品展示'
					}
				}
			},

			// 点击Banner跳转到文章详情
			goToBannerDetail() {
				if (this.bannerArticle && this.bannerArticle.id) {
					uni.navigateTo({
						url: `/subpackages/help/richText/richText?id=${this.bannerArticle.id}`
					})
				}
			},


			// 处理商品数据
			processProductData(data) {
				console.log('开始处理商品数据:', data)

				// 重置数据
				this.clearSkuCache()

				// 设置商品基本信息
				this.productName = data.goodsName || '商品名称'
				this.currentPrice = data.promotionPrice || data.price || 0
				this.originalPrice = data.originPrice || data.price || 0
				this.quantity = 1

				// 设置商品数据
				this.productData = {
					...data,
					quantity: data.quantity || 0,
					skus: data.skus || []
				}

				console.log('设置的商品数据:', {
					productName: this.productName,
					currentPrice: this.currentPrice,
					productData: this.productData
				})

				// 处理商品图片
				this.processProductImages(data)

				// 处理SKU选项
				this.processSkuOptions(data)
			},

			// 处理商品图片
			processProductImages(data) {
				this.productImages = []

				// 处理轮播图
				if (data.goodsImages) {
					try {
						const images = JSON.parse(data.goodsImages)
						if (Array.isArray(images)) {
							this.productImages = images.map(url => ({
								url
							}))
						}
					} catch (e) {
						// 如果解析失败，尝试按逗号分割
						const images = data.goodsImages.split(',').filter(url => url.trim())
						this.productImages = images.map(url => ({
							url: url.trim()
						}))
					}
				}

				// 如果没有轮播图，使用主图
				if (this.productImages.length === 0 && data.goodsImage) {
					this.productImages = [{
						url: data.goodsImage
					}]
				}

				// 如果还是没有图片，保持空数组
				if (this.productImages.length === 0) {
					this.productImages = []
				}
			},

			// 处理SKU选项
			processSkuOptions(data) {
				console.log('开始处理SKU选项')
				console.log('productData:', data)

				if (!data || !data.skus) {
					console.log('没有SKU数据')
					return
				}

				// 清空之前的选项
				this.skuOptions.spec = []

				const skus = data.skus
				const specOptions = data.specOptions

				console.log('原始SKU数据:', skus)
				console.log('规格配置:', specOptions)

				// 解析规格选项配置
				let specConfig = []
				if (specOptions) {
					try {
						specConfig = JSON.parse(specOptions)
						console.log('解析后的规格配置:', specConfig)
					} catch (error) {
						console.error('解析规格配置失败:', error)
						specConfig = []
					}
				}

				// 如果有规格配置，按配置处理
				if (specConfig && specConfig.length > 0) {
					// 使用缓存的SKU规格数据
					const cachedSkus = this.getCachedSkuSpecs()

					// 根据规格配置创建选项
					specConfig.forEach(spec => {
						console.log('处理规格:', spec)
						const specOption = {
							name: spec.attrName,
							values: spec.attrValue,
							selectedValue: spec.attrValue[0], // 默认选择第一个
							skus: []
						}

						// 为每个规格值找到对应的SKU
						spec.attrValue.forEach(value => {
							console.log('查找规格值:', value)
							const matchingSkus = cachedSkus.filter(sku => {
								return sku.parsedSpecs[spec.attrName] === value
							})

							console.log('匹配的SKU:', matchingSkus)

							if (matchingSkus.length > 0) {
								specOption.skus.push({
									skuId: matchingSkus[0].skuId,
									price: matchingSkus[0].price,
									quantity: matchingSkus[0].quantity,
									value: value
								})
							}
						})

						this.skuOptions.spec.push(specOption)
					})
				}

				// 如果有SKU数据，使用第一个SKU的价格作为默认价格
				if (skus.length > 0) {
					this.selectedSkuIndex = 0
				}

				console.log('处理后的SKU选项:', this.skuOptions.spec)
			},

			// 获取缓存的SKU规格数据
			getCachedSkuSpecs() {
				if (!this._skuSpecsCache && this.productData && this.productData.skus) {
					this._skuSpecsCache = this.productData.skus.map(sku => {
						try {
							return {
								...sku,
								parsedSpecs: JSON.parse(sku.fullSpecs || '{}')
							}
						} catch (error) {
							console.error('解析SKU规格失败:', error)
							return {
								...sku,
								parsedSpecs: {}
							}
						}
					})
				}
				return this._skuSpecsCache || []
			},

			// 清理SKU缓存
			clearSkuCache() {
				this._skuSpecsCache = null
				this.skuOptions.spec = []
			},

			// 获取当前选中的SKU
			getSelectedSku() {
				if (!this.productData || !this.productData.skus || this.productData.skus.length === 0) {
					return null
				}

				// 如果没有规格选项，返回第一个SKU
				if (!this.skuOptions.spec || this.skuOptions.spec.length === 0) {
					return this.productData.skus[0]
				}

				// 根据选中的规格找到对应的SKU
				const selectedSpecs = this.skuOptions.spec.map(spec => spec.selectedValue)

				const matchedSku = this.productData.skus.find(sku => {
					try {
						const fullSpecs = JSON.parse(sku.fullSpecs || '{}')
						return selectedSpecs.every((value, index) => {
							const specName = this.skuOptions.spec[index].name
							return fullSpecs[specName] === value
						})
					} catch (error) {
						return false
					}
				})

				// 如果找不到匹配的SKU，返回第一个
				return matchedSku || this.productData.skus[0]
			},

			// 关闭SKU选择器
			closeSkuSelector() {
				this.showSku = false
			},

			// 选择规格值
			selectSpecValue(specIndex, value) {
				this.skuOptions.spec[specIndex].selectedValue = value
				this.updateSelectedSku()
			},

			// 获取SKU价格
			getSkuPrice(specIndex, value) {
				const spec = this.skuOptions.spec[specIndex]
				if (!spec || !spec.skus) return null

				const sku = spec.skus.find(s => s.value === value)
				return sku ? sku.price : null
			},

			// 更新选中的SKU
			updateSelectedSku() {
				// 根据当前选择的规格值找到对应的SKU
				const selectedSpecs = this.skuOptions.spec.map(spec => spec.selectedValue)
				const cachedSkus = this.getCachedSkuSpecs()

				if (cachedSkus && cachedSkus.length > 0) {
					const matchingSku = cachedSkus.find(sku => {
						return selectedSpecs.every((value, index) => {
							const specName = this.skuOptions.spec[index].name
							return sku.parsedSpecs[specName] === value
						})
					})

					if (matchingSku) {
						this.selectedSkuIndex = this.productData.skus.indexOf(matchingSku)
					}
				}
			},

			// 增加数量
			increaseQuantity() {
				this.quantity++
			},

			// 减少数量
			decreaseQuantity() {
				if (this.quantity > 1) {
					this.quantity--
				}
			},

			// 获取SKU弹窗中的商品图片
			getSkuImage() {
				if (this.productImages && this.productImages.length > 0 && this.productImages[0]) {
					return this.productImages[0].url || ''
				}
				return ''
			},

			// 直接添加到购物车（无SKU选择）
			async directAddToCart(data) {
				try {
					// TODO AddCart 会在这里使用
					const params = {
						skuId: data.skus && data.skus.length > 0 ? data.skus[0].skuId : data.goodsId,
						goodsNum: 1
					}
					const res = await AddCart(params)
					if (res.code == 200) {
						uni.showToast({
							title: '已添加到购物车',
							icon: 'success',
							duration: 2000
						})
					} else {
						uni.showToast({
							title: res.msg || '添加失败',
							icon: 'none',
							duration: 2000
						})
					}
				} catch (error) {
					console.error('添加购物车失败:', error)
					uni.showToast({
						title: '添加失败，请重试',
						icon: 'none',
						duration: 2000
					})
				}
			},

			// 确认SKU选择操作
			async confirmSkuAction() {
				if (this.skuActionType === 'cart') {
					await this.confirmAddToCart()
				}
				// 不需要立即购买逻辑，按要求移除
			},

			// 确认添加到购物车
			async confirmAddToCart() {
				try {
					const selectedSku = this.getSelectedSku()
					if (!selectedSku) {
						uni.showToast({
							title: '请选择商品规格',
							icon: 'none',
							duration: 2000
						})
						return
					}

					// TODO AddCart 会在这里使用
					const params = {
						skuId: selectedSku.skuId,
						goodsNum: this.quantity
					}
					const res = await AddCart(params)
					if (res.code == 200) {
						uni.showToast({
							title: '已添加到购物车',
							icon: 'success',
							duration: 2000
						})
						this.closeSkuSelector()
					} else {
						uni.showToast({
							title: res.msg || '添加失败',
							icon: 'none',
							duration: 2000
						})
					}
				} catch (error) {
					console.error('添加购物车失败:', error)
					uni.showToast({
						title: '添加失败，请重试',
						icon: 'none',
						duration: 2000
					})
				}
			},

			// 排序方法
			sortBy(type) {
				if (type === 'price') {
					this.togglePriceSort();
					return;
				}

				this.currentSort = type;
				this.applySorting([...this.originalGoodsList]);
			},

			// 切换价格排序
			togglePriceSort() {
				if (this.currentSort === 'price') {
					// 如果当前已是价格排序，切换升降序
					this.sortOrder = this.sortOrder === 'asc' ? 'desc' : 'asc';
				} else {
					// 如果当前不是价格排序，设置为价格排序，默认降序
					this.currentSort = 'price';
					this.sortOrder = 'desc';
				}
				this.applySorting([...this.originalGoodsList]);
			},

			// 应用排序
			applySorting(goodsList) {
				let sortedList = [...goodsList];

				switch (this.currentSort) {
					case 'price':
						sortedList.sort((a, b) => {
							const priceA = a.price;
							const priceB = b.price;
							return this.sortOrder === 'asc' ? priceA - priceB : priceB - priceA;
						});
						break;
					case 'sales':
						sortedList.sort((a, b) => {
							const salesA = a.virtualSoldNum;
							const salesB = b.virtualSoldNum;
							return salesB - salesA; // 销量总是降序
						});
						break;
					case 'default':
					default:
						// 默认排序：热门商品优先，然后按原始顺序
						sortedList.sort((a, b) => {
							if (a.isHot && !b.isHot) return -1;
							if (!a.isHot && b.isHot) return 1;
							return 0; // 保持原有顺序
						});
						break;
				}

				this.currentGoodsList = sortedList;
			}
		}
	}
</script>

<style lang="scss" scoped>
	.page {
		height: 100vh;
		background: #ffffff;
		overflow: hidden;
	}

	/* 顶部固定栏 */
	.nav-fixed {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		z-index: 999;
		box-shadow: 0 2rpx 8rpx rgba(0, 0, 0, 0.04);
	}

	.nav-inner {
		height: 100%;
		display: flex;
		align-items: flex-end;
		/* 让标题贴近底部，下面还有更多的 padding */
		justify-content: center;
		padding: 0 30rpx 16rpx 30rpx;
		position: relative;
		margin-bottom: 30rpx;
		/* 增加下边距 */
	}

	.nav-title {
		font-size: 32rpx;
		font-weight: 600;
		color: #111;
		line-height: 1;
	}

	/* 页面主体（垂直布局：搜索条 + 分栏区） */
	.page-body {
		display: flex;
		flex-direction: column;
		height: 100%;
	}

	/* 搜索条 */
	.search-wrap {
		padding: 0 12rpx;
		margin-top: 12rpx;
	}

	.search-box {
		height: 80rpx;
		border-radius: 40rpx;
		background: #f6f7f9;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.icon-search {
		font-size: 26rpx;
		margin-right: 8rpx;
	}

	.search-text {
		font-size: 26rpx;
		color: #999;
	}

	/* 排序区域 */
	.sort-wrap {
		display: flex;
		align-items: center;
		background: #fff;
		padding: 0 12rpx;
		border-bottom: 1rpx solid #eee;
	}

	.sort-item {
		flex: 1;
		height: 80rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		position: relative;
		font-size: 26rpx;
		color: #666;
		transition: color 0.3s;

		&.active {
			color: #07c160;
			font-weight: 600;
		}

		&.price-sort {
			flex-direction: row;
			padding: 10rpx 0;
			gap: 8rpx;

			.sort-arrows {
				display: flex;
				flex-direction: column;
				height: 24rpx;
				justify-content: space-between;
			}
		}

		text {
			font-size: 26rpx;
		}
	}

	/* 左右分栏容器 */
	.goods-wrap {
		flex: 1;
		display: flex;
		min-height: 0;
		/* 关键：让子元素 scroll-view 正常占满并可滚动 */
	}

	/* 左侧 Tab */
	.left-tabs {
		width: 176rpx;
		background: #f7f8fa;
		border-right: 1rpx solid #eee;
		height: 100%;
	}

	.tab-item {
		height: 96rpx;
		padding: 0 16rpx;
		display: flex;
		align-items: center;
		color: #666;
		font-size: 26rpx;
	}

	.tab-item.active {
		color: #07c160;
		font-weight: 600;
		background: #ffffff;
		border-left: 6rpx solid #07c160;
	}

	/* 右侧列表 */
	.right-list {
		flex: 1;
		height: 100%;
		background: #fff;
		overflow: hidden;
		/* 确保容器本身不滚动，只有内部scroll-view滚动 */
	}

	/* 商品内容容器 */
	.goods-content {
		padding-bottom: 120rpx;
		/* 大幅增加底部内边距，确保最后一个商品完整显示 */
		min-height: calc(100% + 120rpx);
		/* 确保内容高度足够 */
	}

	/* 顶部Banner容器 */
	.banner-container {
		height: 226rpx;
		margin: 12rpx;
		border-radius: 12rpx;
		overflow: hidden;
		position: relative;
		box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.1);

		&:active {
			transform: scale(0.98);
			transition: transform 0.1s ease;
		}
	}

	.banner-image {
		width: 100%;
		height: 100%;
		object-fit: cover;
	}

	.banner-overlay {
		position: absolute;
		bottom: 0;
		left: 0;
		right: 0;
		background: linear-gradient(transparent, rgba(0, 0, 0, 0.6));
		padding: 30rpx 24rpx 20rpx;
		display: flex;
		justify-content: flex-start;
		align-items: flex-end;
	}

	.banner-title {
		color: #fff;
		font-size: 28rpx;
		font-weight: 600;
		text-shadow: 0 2rpx 4rpx rgba(0, 0, 0, 0.5);
		line-height: 1.4;
		display: -webkit-box;
		-webkit-line-clamp: 2;
		-webkit-box-orient: vertical;
		overflow: hidden;
	}

	/* 默认占位 banner */
	.banner-placeholder {
		height: 226rpx;
		margin: 12rpx;
		border-radius: 12rpx;
		background: linear-gradient(90deg, #f5f7fb, #eef1f6);
	}

	/* 商品项 */
	.item {
		display: flex;
		align-items: stretch;
		padding: 16rpx 12rpx;
		border-bottom: 1rpx solid #f2f2f2;

		&:last-child {
			border-bottom: none;
			padding-bottom: 80rpx;
			/* 大幅增加底部内边距，确保完整显示 */
			margin-bottom: 40rpx;
			/* 额外添加底部外边距 */
		}
	}

	.thumb {
		width: 120rpx;
		height: 120rpx;
		margin-right: 12rpx;
		border-radius: 8rpx;
		background: #f2f2f2;
		position: relative;
		overflow: hidden;
	}

	.thumb-image {
		width: 100%;
		height: 100%;
		object-fit: cover;
	}

	.image-skeleton {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		background: linear-gradient(90deg, #f0f4f8 25%, #e2e8f0 50%, #f0f4f8 75%);
		background-size: 200% 100%;
		animation: skeleton-loading 1.8s infinite ease-in-out;
	}

	.item-badge {
		position: absolute;
		top: 4rpx;
		left: 4rpx;
		z-index: 2;
	}

	.badge-icon {
		width: 32rpx;
		height: 32rpx;
	}

	.info {
		flex: 1;
		display: flex;
		flex-direction: column;
		min-width: 0;
	}

	.title {
		font-size: 28rpx;
		color: #333;
		line-height: 1.4;
		display: -webkit-box;
		-webkit-line-clamp: 2;
		-webkit-box-orient: vertical;
		overflow: hidden;
	}

	.tags {
		margin-top: 6rpx;
		font-size: 22rpx;
		color: #999;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}

	.sales-info {
		display: flex;
		align-items: center;
		margin-top: 6rpx;
		font-size: 22rpx;
	}

	.sales-label {
		color: #666;
		font-size: 22rpx;
	}

	.sales-count {
		color: #e54d42;
		font-weight: 600;
		font-size: 24rpx;
	}

	.bottom {
		margin-top: 10rpx;
		display: flex;
		align-items: center;
		justify-content: space-between;
	}

	.price-section {
		display: flex;
		align-items: center;
		gap: 8rpx;
	}

	.price {
		color: #e54d42;
		font-weight: 600;
		font-size: 28rpx;
	}

	.original-price {
		color: #999;
		font-size: 22rpx;
		text-decoration: line-through;
	}

	.btn-add {
		width: 48rpx;
		height: 48rpx;
		border-radius: 50%;
		color: #fff;
		font-size: 28rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 8rpx;

		&:active {
			transform: scale(0.95);
		}
	}

	.add-cart-icon {
		width: 32rpx;
		height: 32rpx;
	}

	/* 加载状态 */
	.loading-container {
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 60rpx 0;
	}

	.loading-text {
		font-size: 26rpx;
		color: #999;
	}

	/* 空状态 */
	.empty-container {
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 120rpx 0 160rpx 0;
		/* 增加底部内边距 */
	}

	.empty-text {
		font-size: 26rpx;
		color: #999;
	}

	/* 推荐商品区域 */
	.recommend-section {
		margin: 20rpx 12rpx;
		background: #fff;
		border-radius: 12rpx;
		padding: 20rpx;
	}

	.recommend-header {
		margin-bottom: 20rpx;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.recommend-title {
		position: relative;
		padding: 0 20rpx;

		.title-text {
			font-size: 28rpx;
			font-weight: 600;
			color: #333;
		}

		&::before,
		&::after {
			content: '';
			position: absolute;
			top: 50%;
			width: 30rpx;
			height: 2rpx;
			background: linear-gradient(90deg, #07c160, #00b377);
		}

		&::before {
			left: -40rpx;
		}

		&::after {
			right: -40rpx;
		}
	}

	.recommend-grid {
		display: grid;
		grid-template-columns: repeat(2, 1fr);
		gap: 16rpx;
	}

	.recommend-item {
		background: #fff;
		border-radius: 8rpx;
		overflow: hidden;
		border: 1rpx solid #f0f0f0;
		transition: all 0.2s ease;

		&:active {
			transform: scale(0.98);
		}
	}

	.recommend-thumb {
		position: relative;
		height: 160rpx;
		overflow: hidden;
	}

	.recommend-image {
		width: 100%;
		height: 100%;
		object-fit: cover;
	}

	.recommend-badge {
		position: absolute;
		top: 8rpx;
		left: 8rpx;
		background: #FF6B35;
		color: #fff;
		font-size: 18rpx;
		padding: 4rpx 8rpx;
		border-radius: 6rpx;
		font-weight: 600;
	}

	.recommend-info {
		padding: 12rpx;
	}

	.recommend-name {
		font-size: 24rpx;
		color: #333;
		line-height: 1.3;
		margin-bottom: 8rpx;
		display: -webkit-box;
		-webkit-line-clamp: 2;
		-webkit-box-orient: vertical;
		overflow: hidden;
	}

	.recommend-price-row {
		display: flex;
		align-items: center;
		gap: 8rpx;
	}

	.recommend-price {
		font-size: 26rpx;
		color: #e54d42;
		font-weight: 600;
	}

	.recommend-original {
		font-size: 20rpx;
		color: #999;
		text-decoration: line-through;
	}

	/* 骨架屏动画 */
	@keyframes skeleton-loading {
		0% {
			background-position: 200% 0;
		}

		100% {
			background-position: -200% 0;
		}
	}

	/* SKU选择弹窗 */
	.sku-popup {
		padding: 24rpx;
		display: flex;
		flex-direction: column;
		max-height: 80vh;
		min-height: 400rpx;
		box-sizing: border-box;
		background: #fff;
		border-radius: 20rpx 20rpx 0 0;
	}

	.sku-header {
		display: flex;
		align-items: center;
		margin-bottom: 24rpx;
		padding-bottom: 16rpx;
		border-bottom: 1rpx solid #e9ecef;
	}

	.sku-product-image {
		width: 120rpx;
		height: 120rpx;
		border-radius: 8rpx;
		margin-right: 16rpx;
	}

	.sku-product-info {
		flex: 1;
	}

	.sku-product-name {
		font-size: 32rpx;
		color: #333333;
		font-weight: 700;
		margin-bottom: 4rpx;
	}

	.sku-product-price {
		.sku-price-symbol {
			font-size: 28rpx;
			color: #009966;
			font-weight: 600;
		}

		.sku-price-value {
			font-size: 40rpx;
			color: #009966;
			font-weight: 700;
		}
	}

	.sku-close {
		padding: 8rpx;
		border-radius: 50%;
		transition: background-color 0.2s ease;
	}

	.sku-close:active {
		background-color: rgba(0, 0, 0, 0.05);
	}

	.sku-options {
		flex: 1;
		overflow-y: auto;
		-webkit-overflow-scrolling: touch;
		padding-bottom: 20rpx;
		min-height: 200rpx;
	}

	.sku-option-group {
		margin-bottom: 32rpx;
	}

	.option-title {
		font-size: 28rpx;
		color: #333;
		font-weight: 600;
		margin-bottom: 16rpx;
		display: block;
	}

	.option-items {
		display: flex;
		flex-wrap: wrap;
		gap: 12rpx;
	}

	.option-item {
		border: 2rpx solid #e9ecef;
		border-radius: 8rpx;
		padding: 16rpx 20rpx;
		background: #fff;
		cursor: pointer;
		transition: all 0.2s ease;
		min-width: 80rpx;
		text-align: center;
	}

	.option-item.active {
		border-color: #009966;
		background: #f0f9f6;
		color: #009966;
	}

	.option-item:active {
		transform: scale(0.95);
	}

	.option-content {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 4rpx;
	}

	.option-content text {
		font-size: 26rpx;
		line-height: 1.2;
	}

	.sku-price {
		font-size: 22rpx;
		color: #999;
	}

	.option-item.active .sku-price {
		color: #009966;
	}

	.quantity-selector {
		margin-top: 32rpx;
		display: flex;
		align-items: center;
		justify-content: space-between;
	}

	.quantity-title {
		font-size: 28rpx;
		color: #333;
		font-weight: 600;
	}

	.quantity-control {
		display: flex;
		align-items: center;
		border: 2rpx solid #e9ecef;
		border-radius: 8rpx;
		overflow: hidden;
	}

	.quantity-btn {
		width: 64rpx;
		height: 64rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		background: #f8f9fa;
		cursor: pointer;
		transition: background-color 0.2s ease;
	}

	.quantity-btn:active {
		background: #e9ecef;
	}

	.quantity-value {
		width: 80rpx;
		height: 64rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		font-size: 28rpx;
		color: #333;
		font-weight: 600;
		background: #fff;
		border-left: 2rpx solid #e9ecef;
		border-right: 2rpx solid #e9ecef;
	}

	.sku-footer {
		display: flex;
		align-items: center;
		justify-content: space-between;
		padding-top: 24rpx;
		border-top: 2rpx solid #e9ecef;
		margin-top: 24rpx;
	}

	.sku-total {
		display: flex;
		align-items: center;
		gap: 8rpx;
	}

	.sku-total text {
		font-size: 28rpx;
		color: #333;
	}

	.total-price {
		font-size: 36rpx;
		color: #009966;
		font-weight: 700;
	}

	.sku-action {
		background: linear-gradient(135deg, #009966, #00b377);
		color: #fff;
		padding: 24rpx 48rpx;
		border-radius: 48rpx;
		font-size: 28rpx;
		font-weight: 600;
		cursor: pointer;
		transition: all 0.2s ease;
		box-shadow: 0 4rpx 12rpx rgba(0, 153, 102, 0.3);
	}

	.sku-action:active {
		transform: scale(0.95);
		box-shadow: 0 2rpx 8rpx rgba(0, 153, 102, 0.3);
	}

	.sku-action text {
		color: #fff;
	}
</style>