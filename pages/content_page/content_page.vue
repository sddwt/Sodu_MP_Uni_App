<template>
	<view>
		<view class="root" :style="contentStyle" @click="handleSettingClick">
			<view class="nav-bar-container">
				<uniNavBar :backgroundColor="config.bgColor" shadow="false">
					<view class="title-container">
						<view class="book-name">
							{{book.name}}
						</view>
						<view class="catalog-name">
							{{currentCatalog ? currentCatalog.name : ''}}
						</view>
					</view>
				</uniNavBar>
			</view>
			<view class="outer">
				<scrollContent :book="book" :content="currentCatalog.content" :height="`calc(100vh - ${navBarHeight}px - 20px)`"
				 @switchAction="handleSwitchAction"></scrollContent>
				<!-- <swiperContent v-if="config.readType === 2" :config="config" :book="book" :content="currentCatalog.content"
				 @switchAction="handleSwitchAction" /> -->
			</view>
			<view class="bottom"></view>
			<Setting v-if="showSettingPanel" :isExistShelf="isExistShelf" :book="book" @close="handleCloseSettingPanel"
			 @updateConfig="handleUpdateConfig" @switchAction="handleSwitchAction" @addBookToShelf="checkBookStatus" @reload="handleReload" />

		</view>
		<Loading v-if="isLoading" class="loading-container" text="加载中..." mask="true" click="false"></Loading>
	</view>
</template>

<script>
	import uniNavBar from '@/components/uni-nav-bar/uni-nav-bar.vue';
	import scrollContent from '../../components/scroll-content/scroll-content.vue'
	// import swiperContent from '../../components/swiper-content/swiper-content.vue'
	import Setting from '../../components/content-setting-panel/content-setting-panel.vue'
	import {
		getContent,
		getBookInfo
	} from '@/api/content.js'
	import {
		getConfig,
		colors
	} from '../../utils/config.js'
	import {
		decodeUTF8,
		encodeUTF8
	} from '../../utils/encode.js'
	import {
		getShelfBooks,
		getShelfBooksById
	} from '../../utils/bookShelf.js'
	import {
		mapState,
		mapMutations
	} from 'vuex'
	export default {
		components: {
			uniNavBar,
			Setting,
			scrollContent,
			// swiperContent
		},
		data() {
			return {
				navBarHeight: 44 + uni.getSystemInfoSync().statusBarHeight,
				clientWidth: uni.getSystemInfoSync().screenWidth,
				clientHeight: uni.getSystemInfoSync().screenHeight,
				showSettingPanel: false,
				isLoading: false,
				isRequestBookInfo: false,
				book: null,
				config: null,
				currentCatalog: null,
				bookInfo: null,
				showBookInfoPanel: false,
				isExistShelf: false
			};
		},
		computed: {
			contentStyle() {
				let themeValue = colors[this.config.theme]
				let style =
					`color: ${themeValue.color};` +
					`background: ${themeValue.bgColor};` +
					`line-height: ${this.config.lineHeight};` +
					`font-size: ${this.config.fontSize}px;`
				return style
			},
			...mapState(['storeBookInfo'])
		},
		created(e) {
			uni.$on('navigateToCatalog', this.handleSwitchCatalog)
			this.config = getConfig()
		},
		mounted() {},
		onLoad(option) {
			if (this.$store.state.status === 0) {
				uni.reLaunch({
					url: '../home_page_simple/home_page_simple'
				})
				return
			}
			if (option.from === 'info') {
				this.setBookInfo(null)
			}
			if (!option.book) {
				return
			}
			this.book = JSON.parse(decodeUTF8(option.book))
			// // if (!option.book) {
			// this.book = {
			// 	bookId: "729376",
			// 	chapterName: "第三百二十六章 野心勃勃",
			// 	lyWeb: "乐安宣书网",
			// 	name: "我的1979",
			// 	soduUpdatePageUrl: "https://www.sodu2020.com/mulu_729376.html",
			// 	sourceUrl: "https://www.biquge5.tv/90_90691/31654669.html",
			// 	updateTime: "2020/03/31 06:16"
			// }
			// // } else {
			// // }
			this.initData()
		},
		onUnload() {
			uni.$off('navigateToCatalog')
			this.clearStorage()
		},
		methods: {
			...mapMutations(['setBookInfo']),
			initData() {
				this.checkBookStatus()
				this.requestCurrentCatalog({
					name: this.book.chapterName,
					url: this.book.sourceUrl,
					content: ''
				})
				if (!this.storeBookInfo) {
					// 请求目录
					this.requestBookInfo(this.book.sourceUrl)
				} else {
					this.bookInfo = this.storeBookInfo
				}
			},
			// 检查是否已经在书架中
			checkBookStatus() {
				if (!this.book) {
					return
				}
				getShelfBooks((books) => {
					let index = books.findIndex(e => e.bookId === this.book.bookId)
					this.isExistShelf = index === -1 ? false : true
				})
			},
			// 加载当前章节数据
			async requestCurrentCatalog(item, time = 0) {
				if (this.isLoading) {
					return
				}
				try {
					if (item.content) {
						this.currentCatalog = item
						this.updateBookShelfInfo(item)
					} else {
						this.isLoading = true
						let res = await this.requestContent(item.url)
						if (res) {
							item.content = res
							this.currentCatalog = item
							this.formatCurrentCatalog()
							this.updateBookShelfInfo(Object.assign({}, item, {
								lyWeb: this.book.lyWeb
							}))
						} else {
							throw new Error()
						}
					}
				} catch (error) {
					if (time < 3) {
						this.requestCurrentCatalog(item, ++time)
					}
				} finally {
					this.isLoading = false
				}
			},
			updateBookShelfInfo(item) {
				if (!this.isExistShelf) {
					return
				}
				uni.$emit('updateShelfBookMark', Object.assign({}, item, {
					id: this.book.bookId
				}))
			},
			async requestContent(url) {
				try {
					let res = await getContent(url)
					if (res.code === 0) {
						return res.result
					} else {
						return null
					}
				} catch (e) {
					return null
				}
			},
			// 加载目录数据
			async requestBookInfo(url, time = 0) {
				try {
					this.isRequestBookInfo = true
					let res = await getBookInfo(url)
					if (res.code === 0) {
						this.bookInfo = res.result
						this.isRequestBookInfo = false
						this.setBookInfo(res.result)
						this.formatCurrentCatalog()
					} else {
						throw new Error()
					}
				} catch (e) {
					console.log(e)
					if (time < 3) {
						this.requestBookInfo(url, ++time)
					} else {
						this.bookInfo = null
						this.isRequestBookInfo = false
						uni.showToast({
							title: ' 目录加载失败',
							icon: 'none',
							duration: 3000
						})
					}
				}
			},
			formatCurrentCatalog() {
				try {
					if (!this.bookInfo || !this.currentCatalog) {
						return
					}
					if (this.currentCatalog.content) {
						let index = this.bookInfo.catalogs.findIndex(e => e.url === this.currentCatalog.url)
						if (index > -1) {
							this.bookInfo.catalogs[index].content = this.currentCatalog.content
							this.currentCatalog.index = index
						}
					}
				} catch (e) {
					console.log(e)
				}

			},
			// 预加载接下来2章内容
			preLoadNextCatalogs() {
				let index = this.bookInfo.catalogs.findIndex(e => e.url === this.currentCatalog.url)
				if (index === -1) {
					return
				}
				if (index < this.bookInfo.catalogs.length - 1) {
					let list = this.bookInfo.catalogs.slice(index + 1, index + 3)
					list.forEach(e => {
						if (!e.content) {
							this.requestContent(e.url).then(res => {
								e.content = res
							})
						}
					})
				}
			},
			handleSettingClick(e) {
				if (e.detail.x < this.clientWidth * 1 / 4 || e.detail.x > this.clientWidth * 3 / 4) {
					return
				}
				if (e.detail.y < this.clientHeight * 1 / 5 || e.detail.y > this.clientHeight * 4 / 5) {
					return
				}
				this.showSettingPanel = true
			},
			handleCloseSettingPanel() {
				this.showSettingPanel = false
			},
			handleUpdateConfig(newValue) {
				this.config = newValue
			},
			//  1：上一章  2： 目录 3：下一章 4：更新章节
			handleSwitchAction(key) {
				this.handleCloseSettingPanel()
				if (this.isLoading) {
					return
				}
				if (key < 4 && !this.bookInfo) {
					uni.showToast({
						title: this.isRequestBookInfo ? '数据请求中，请稍候' : '无章节数据',
						icon: 'none',
						duration: 2000
					})
					return
				}
				switch (key) {
					case 1:
						this.handlePreChapter()
						break;
					case 2:
						this.goToBookInfoPage()
						break;
					case 3:
						this.handleNextChapter()
						break;
					case 4:
						this.goToUpdatePage()
						break;
				}
			},
			goToUpdatePage() {
				let url = '../../pages/sodu_update_sites/sodu_update_sites'
				uni.redirectTo({
					url: url + `?book=${encodeUTF8(JSON.stringify(this.book))}`,
					animationType: 'pop-in',
					animationDuration: 200
				})
			},
			// 打开目录
			goToBookInfoPage() {
				let url = '../../pages/book_info_page/book_info_page'
				uni.navigateTo({
					url: url + `?book=${encodeUTF8(JSON.stringify(this.book))}`,
					animationType: 'pop-in',
					animationDuration: 200
				})
			},
			// 上一章
			handlePreChapter() {
				let index = this.currentCatalog.index
				if (index === 0 || typeof index === 'undefined') {
					this.goToBookInfoPage()
				} else {
					let item = this.bookInfo.catalogs[index - 1]
					this.handleSwitchCatalog(item)
				}
			},
			// 下一章
			handleNextChapter() {
				let index = this.currentCatalog.index
				if (index === this.bookInfo.catalogs.length - 1 || typeof index === 'undefined') {
					this.goToBookInfoPage()
				} else {
					this.handleSwitchCatalog(this.bookInfo.catalogs[index + 1])
					this.preLoadNextCatalogs()
				}
			},
			//切换章节
			handleSwitchCatalog(item) {
				if (item) {
					let temp = this.bookInfo.catalogs[item.index]
					this.requestCurrentCatalog(temp)
				}
			},
			handleReload() {
				this.requestCurrentCatalog({
					index: this.currentCatalog.index,
					url: this.currentCatalog.url,
					name: this.currentCatalog.name,
					content: null
				})
			},
			clearStorage() {
				this.setBookInfo(null)
			}
		}
	}
</script>
<style lang="less" scoped>
	.root {
		position: fixed;
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100vh;
		overflow-x: hidden;
		overflow-y: hidden;
		display: flex;
		flex-direction: column;
		z-index: 99;

		.title-container {
			line-height: 40upx;
			display: flex;
			flex-direction: column;
			overflow: hidden;
			width: 45vw;

			.book-name {
				flex: 1;
				font-weight: 600;
				text-align: center;
				font-size: 28upx;
				overflow: hidden;
				word-break: keep-all;
				/* 不换行 */
				white-space: nowrap;
				/* 不换行 */
				overflow: hidden;
				/* 内容超出宽度时隐藏超出部分的内容 */
				text-overflow: ellipsis;
			}

			.catalog-name {
				flex: 1;
				font-weight: lighter;
				text-align: center;
				font-size: 24upx;
				overflow: hidden;
				word-break: keep-all;
				/* 不换行 */
				white-space: nowrap;
				/* 不换行 */
				overflow: hidden;
				/* 内容超出宽度时隐藏超出部分的内容 */
				text-overflow: ellipsis;
			}
		}

		.outer {
			flex: 1;
			overflow-x: auto;
		}

		.bottom {
			width: 100vw;
			height: 20px;
			font-size: 24upx;
			display: flex;
			align-items: center;
			padding-left: 30upx;
			background: transparent;
		}
	}
</style>
