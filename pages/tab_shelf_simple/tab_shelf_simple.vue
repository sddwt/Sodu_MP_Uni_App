<template>
	<view class="tab-shelf">
		<mescroll-uni :fixed="false" class="scroll" ref="mescrollRef" @init="mescrollInit" @down="downCallback" :down="downOption"
		 :up="upOption">
			<view v-if="books && books.length > 0">
				<BookItem v-for="(item) in books" :key="item.bookId" :book="item" isShelf="true" @itemLongPress="handleItemLongPress" />
			</view>
			<view v-else class="empty">
				您的收藏夹很孤独, 请搜索关键词后长按列表项添加收藏~。
			</view>
		</mescroll-uni>
		<Popupmenus v-if="showMenu" :book="selectedBook" @closeMenu="closeMenu" @bookRefresh="bookRefresh" type="shelf_simple"
		 @deleteBook="handleDeleteBook" />
	</view>
</template>

<script>
	import {
		getShelfBooks,
		addToShelf,
		updateBook,
		deleteBook
	} from '@/utils/bookShelfSimple.js'
	import {
		getUpdateSites
	} from '../../api/sodu.js'

	import BookItem from '@/components/book-item-simple/book-item-simple.vue'
	import Popupmenus from '@/components/PopupMenus/index.vue'
	import MescrollMixin from "mescroll-uni/mescroll-mixins.js";
	export default {
		mixins: [MescrollMixin],
		components: {
			BookItem,
			Popupmenus
		},
		props: {
			visiable: {
				type: Boolean,
				default: false
			}
		},
		data() {
			return {
				books: [],
				showMenu: false,
				selectedBook: null,
				// 下拉刷新的常用配置
				downOption: {
					use: true, // 是否启用下拉刷新; 默认true
					auto: false, // 是否在初始化完毕之后自动执行下拉刷新的回调; 默认true
					textLoading: '开始刷新...'
				},
				upOption: {
					use: false
				}
			};
		},
		created() {
			this.initEventBus()
			this.initBooks()
		},
		mounted() {
			this.handlePullDwonRefresh()
		},
		destroyed() {
			this.removeEventBus()
		},
		methods: {
			initEventBus() {
				//  添加至书架
				uni.$on('addBookToShelf', this.addBook)
				//  跟新书架所有书
				uni.$on('updateBookShelf', this.initBooks)
				//  更新单本书
				uni.$on('updateShelfBook', this.updateShelfBook)
				//  更新阅读记录
				uni.$on('updateShelfBookMark', this.updateShelfBookMark)
				// 下拉刷新
				uni.$on('pullDwomRefresh', this.handlePullDwonRefresh)
			},
			removeEventBus() {
				uni.$off('addBookToShelf')
				uni.$off('updateBookShelf')
				uni.$off('updateShelfBook')
				uni.$off('updateShelfBookMark')
			},
			handlePullDwonRefresh() {
				this.books.forEach(e => {
					this.bookRefresh(e.bookId)
				})
			},
			addBook(e) {
				e.isLoading = false
				addToShelf(e, () => {
					this.books.unshift(e)
					this.handleRefreshBook(e)
					uni.showToast({
						title: '添加成功',
						duration: 2000,
						icon: 'success'
					})
				})
			},
			initBooks() {
				getShelfBooks((books) => {
					if (books) {
						let temp = books.map(e => {
							e.isLoading = false
							return e
						})
						temp.sort(function(a, b) {
							return b.shelfTime - a.shelfTime
						})
						this.books = temp
					}
				})
			},
			bookRefresh(id) {
				let index = this.books.findIndex(e => e.bookId === id)
				if (index > -1) {
					this.handleRefreshBook(this.books[index])
				}
				this.closeMenu()
			},
			async handleRefreshBook(book) {
				try {
					if (!book.soduUpdatePageUrl || book.isLoading || book.type === 1) {
						return
					}
					book.isLoading = true
					let result = await this.requestBookInfo(book.soduUpdatePageUrl, book.bookId, 1)
					if (!result || result.code !== 0 || result.books.length === 0) {
						return
					}
					book.chapterName = result.books[0].chapterName
					this.updateShelfBook(book)
				} catch (e) {
					console.log(e)
				} finally {
					book.isLoading = false
				}
			},
			// 加载目录数据
			async requestBookInfo(url, bookId, page, time = 0) {
				try {
					this.isRequestBookInfo = true
					let res = await getUpdateSites(url, bookId, page)
					if (res.code === 0) {
						return res.result
					} else {
						throw new Error()
					}
				} catch (e) {
					if (time < 3) {
						this.requestBookInfo(url, ++time)
					} else {
						return null
					}
				}
			},
			handleDeleteBook(id) {
				let index = this.books.findIndex(e => e.bookId === id)
				if (index > -1) {
					deleteBook(id, () => {
						this.books.splice(index, 1)
						this.closeMenu()
					})
				}
			},
			updateShelfBook(book) {
				updateBook(book)
				this.books.sort((a, b) => {
					return b.shelfTime - a.shelfTime
				})
			},
			handleItemLongPress(item) {
				this.selectedBook = item
				this.showMenu = true
			},
			closeMenu() {
				this.showMenu = false
				this.selectedBook = null
			},
			handleBookClick(book) {
				setTimeout(() => {
					book.hasNew = false
					book.userlastCatalogsUrl = book.lastChapterUrl
					book.shelfTime = +new Date()
					this.updateShelfBook(book)
				}, 300)
			},
			downCallback() {
				this.handlePullDwonRefresh()
				setTimeout(() => {
					this.mescroll.endSuccess()
				}, 800)
			}
		}
	}
</script>


<style lang="less" scoped>
	.tab-shelf {
		position: relative;
		min-height: 50vh;
		box-sizing: border-box;
		height: 100%;
		flex: 1;
		padding-bottom: 10upx;

		.empty {
			position: fixed;
			width: 100vw;
			left: 0;
			top: 40%;
			font-weight: 34upx;
			color: #808080;
			text-align: center;
			padding: 0 20%;
			box-sizing: border-box;
			line-height: 1.5em;
			height: 40vh;
		}
	}
</style>
