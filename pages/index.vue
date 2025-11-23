<template>
	<div class="app">
		<!-- AWAっぽいヒーロー背景 -->
		<header class="hero">
			<div class="hero-inner">
				<div class="hero-text">
					<p class="hero-tag">CHANNEL ANALYTICS</p>
					<h1 class="hero-title">YouTube コメント ジェリエル作！</h1>
					<p class="hero-sub">
						任意の YouTube 動画の <span>videoId</span> を入力すると、
						その動画のコメントを一覧で確認できます。
					</p>

					<div class="hero-actions">
						<NuxtLink to="/top-videos" class="hero-link">
							チャンネル内コメント数 TOP10 ビューアへ
						</NuxtLink>
					</div>
				</div>
			</div>
		</header>

		<main class="main">
			<!-- 入力カード -->
			<section class="card card-input fade-up">
				<h2 class="card-title">コメントを取得</h2>

				<label class="label" for="video-id-input">YouTube Video ID</label>
				<div class="input-wrap">
					<input
						id="video-id-input"
						v-model="videoId"
						class="input"
						placeholder="例: dQw4w9WgXcQ（URLの v= の後ろ）"
					/>
				</div>

				<p class="hint">
					例）https://www.youtube.com/watch?v=<strong>dQw4w9WgXcQ</strong> →
					<span class="highlight">dQw4w9WgXcQ</span> を入力
				</p>

				<div class="btn-row">
					<button
						class="btn"
						@click="fetchComments"
						:disabled="loading || !videoId"
					>
						<span v-if="!loading">コメント取得</span>
						<span v-else> <span class="spinner"></span> 読み込み中... </span>
					</button>

					<!-- ★ Clearボタン -->
					<button
						type="button"
						class="btn ghost"
						@click="clearAll"
						:disabled="loading && !videoId && !comments.length"
					>
						Clear
					</button>
				</div>

				<p v-if="errorMessage" class="error">{{ errorMessage }}</p>
			</section>

			<!-- コメント一覧 -->
			<section
				v-if="comments.length"
				class="card card-comments fade-up delay-1"
			>
				<h2 class="card-title">コメント一覧</h2>

				<!-- ソート / フィルタ -->
				<div class="controls">
					<div class="control-group">
						<label class="control-label">並び順</label>
						<select v-model="sortKey" class="select">
							<option value="relevance">関連度（デフォルト）</option>
							<option value="newest">新しい順</option>
							<option value="oldest">古い順</option>
							<option value="likes">高評価が多い順</option>
						</select>
					</div>

					<div class="control-group wide">
						<label class="control-label">キーワードフィルタ</label>
						<input
							v-model="filterKeyword"
							class="input small"
							placeholder="例: thanks / ありがとう"
						/>
					</div>
				</div>

				<p class="count">
					全 {{ comments.length }} 件中 {{ filteredCount }} 件を表示中（{{
						currentPage
					}}
					/ {{ totalPages }} ページ）
				</p>

				<!-- アニメーション付きリスト -->
				<transition-group name="comment-list" tag="ul" class="comment-list">
					<li
						v-for="(c, i) in visibleComments"
						:key="`${c.publishedAt}-${i}`"
						class="comment-item"
					>
						<div class="comment-header">
							<img
								v-if="c.authorProfileImageUrl"
								:src="c.authorProfileImageUrl"
								class="avatar"
							/>

							<div class="comment-author">
								<a
									v-if="c.authorChannelUrl"
									:href="c.authorChannelUrl"
									target="_blank"
									rel="noopener noreferrer"
									class="author-name"
								>
									{{ c.author }}
								</a>
								<span v-else class="author-name">{{ c.author }}</span>
								<span class="date">{{ formatDate(c.publishedAt) }}</span>
							</div>

							<span class="likes">👍 {{ c.likeCount }}</span>
						</div>

						<p class="comment-body" v-html="c.textHtml" />
					</li>
				</transition-group>

				<!-- ページネーション -->
				<div class="pagination" v-if="totalPages > 1">
					<button
						class="pager-btn"
						type="button"
						:disabled="currentPage === 1"
						@click="currentPage--"
					>
						前へ
					</button>
					<span class="page-info">{{ currentPage }} / {{ totalPages }}</span>
					<button
						class="pager-btn"
						type="button"
						:disabled="currentPage === totalPages"
						@click="currentPage++"
					>
						次へ
					</button>
				</div>

				<!-- API 追加読み込み -->
				<div class="more-wrap" v-if="nextPageToken">
					<button @click="loadMore" class="btn ghost" :disabled="loading">
						<span v-if="!loading">YouTube からさらに読み込む</span>
						<span v-else>
							<span class="spinner small"></span> 読み込み中...
						</span>
					</button>
				</div>
			</section>

			<section
				v-else-if="!loading && hasFetchedOnce"
				class="card card-empty fade-up delay-1"
			>
				コメントが見つかりませんでした。
			</section>
		</main>
	</div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

// APIキー（ローカルでの利用想定）
const apiKey = 'AIzaSyByrdYL7fducADQbNf_1CVXp2muhroW690'

const videoId = ref('')
const comments = ref([])
const nextPageToken = ref(null)
const loading = ref(false)
const errorMessage = ref('')
const hasFetchedOnce = ref(false)

// ソート / フィルタ / ページネーション用
const sortKey = ref('relevance') // relevance | newest | oldest | likes
const filterKeyword = ref('')
const currentPage = ref(1)
const pageSize = 20

// APIコール
const callYouTubeCommentsApi = async (options = {}) => {
	const params = new URLSearchParams({
		key: apiKey,
		part: 'snippet',
		maxResults: '100',
		order: 'relevance',
	})

	if (options.videoId) params.set('videoId', options.videoId)
	if (options.pageToken) params.set('pageToken', options.pageToken)

	const url =
		'https://www.googleapis.com/youtube/v3/commentThreads?' + params.toString()
	const res = await fetch(url)

	if (!res.ok) {
		const text = await res.text()
		throw new Error(`YouTube API error: ${res.status} ${text}`)
	}

	const data = await res.json()

	const mappedComments =
		data.items?.map((item) => {
			const s = item.snippet.topLevelComment.snippet
			const authorChannelId = s.authorChannelId?.value

			return {
				author: s.authorDisplayName,
				authorProfileImageUrl: s.authorProfileImageUrl || null,
				authorChannelUrl: authorChannelId
					? `https://www.youtube.com/channel/${authorChannelId}`
					: null,
				textHtml: s.textDisplay,
				textPlain: s.textOriginal ?? s.textDisplay ?? '',
				likeCount: s.likeCount,
				publishedAt: s.publishedAt,
			}
		}) ?? []

	return {
		comments: mappedComments,
		nextPageToken: data.nextPageToken ?? null,
	}
}

const fetchComments = async () => {
	if (!videoId.value) {
		errorMessage.value = 'Video ID を入力してください'
		return
	}

	loading.value = true
	errorMessage.value = ''
	hasFetchedOnce.value = false
	comments.value = []
	nextPageToken.value = null
	currentPage.value = 1

	try {
		const result = await callYouTubeCommentsApi({
			videoId: videoId.value.trim(),
		})
		comments.value = result.comments
		nextPageToken.value = result.nextPageToken
		hasFetchedOnce.value = true
	} catch (e) {
		errorMessage.value = e.message || 'コメント取得に失敗しました'
	} finally {
		loading.value = false
	}
}

const loadMore = async () => {
	if (!videoId.value || !nextPageToken.value) return

	loading.value = true
	errorMessage.value = ''

	try {
		const result = await callYouTubeCommentsApi({
			videoId: videoId.value.trim(),
			pageToken: nextPageToken.value,
		})
		comments.value.push(...result.comments)
		nextPageToken.value = result.nextPageToken
		// 新規読み込み後は最後のページにジャンプ
		currentPage.value = Math.ceil(filteredCount.value / pageSize)
	} catch (e) {
		errorMessage.value = e.message || '追加読み込みに失敗しました'
	} finally {
		loading.value = false
	}
}

const clearAll = () => {
	videoId.value = ''
	comments.value = []
	nextPageToken.value = null
	hasFetchedOnce.value = false
	errorMessage.value = ''
	filterKeyword.value = ''
	sortKey.value = 'relevance'
	currentPage.value = 1
}

// ソート＋フィルタ済み配列
const filteredSortedComments = computed(() => {
	let arr = comments.value.slice()

	// ソート
	if (sortKey.value === 'newest') {
		arr.sort(
			(a, b) =>
				new Date(b.publishedAt).getTime() - new Date(a.publishedAt).getTime()
		)
	} else if (sortKey.value === 'oldest') {
		arr.sort(
			(a, b) =>
				new Date(a.publishedAt).getTime() - new Date(b.publishedAt).getTime()
		)
	} else if (sortKey.value === 'likes') {
		arr.sort((a, b) => (b.likeCount ?? 0) - (a.likeCount ?? 0))
	} // relevance は API の順序そのまま

	// フィルタ
	const kw = filterKeyword.value.trim().toLowerCase()
	if (kw) {
		arr = arr.filter((c) => {
			const text = (c.textPlain || '').toLowerCase()
			const author = (c.author || '').toLowerCase()
			return text.includes(kw) || author.includes(kw)
		})
	}

	return arr
})

const filteredCount = computed(() => filteredSortedComments.value.length)

const totalPages = computed(() =>
	filteredCount.value === 0 ? 1 : Math.ceil(filteredCount.value / pageSize)
)

const visibleComments = computed(() => {
	const start = (currentPage.value - 1) * pageSize
	return filteredSortedComments.value.slice(start, start + pageSize)
})

// フィルタ条件が変わったら 1 ページ目に戻す
watch([filterKeyword, sortKey, comments], () => {
	currentPage.value = 1
})

const formatDate = (iso) => {
	const d = new Date(iso)
	return Number.isNaN(d.getTime()) ? iso : d.toLocaleString()
}
</script>

<style scoped>
:global(body) {
	margin: 0;
	background: #050509;
}

/* 全体背景：グレー寄りの強めグラデーション */
.app {
	min-height: 100vh;
	background: radial-gradient(
			circle at top left,
			#3b3f46 0,
			#181b20 40%,
			#050608 75%
		)
		fixed;
	color: #e5e7eb;
	font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI',
		sans-serif;
	animation: app-fade-in 0.4s ease-out;
}

/* ヒーロー（上の帯部分） */
.hero {
	padding: 32px 16px 24px;
}

.hero-inner {
	max-width: 960px;
	margin: 0 auto;
}

.hero-text {
	max-width: 520px;
}

.hero-tag {
	font-size: 0.7rem;
	letter-spacing: 0.16em;
	text-transform: uppercase;
	color: #9ca3af;
	margin-bottom: 6px;
}

.hero-title {
	font-size: 2rem;
	font-weight: 700;
	margin-bottom: 8px;
	color: #f9fafb;
}

.hero-sub {
	font-size: 0.9rem;
	color: #d1d5db;
	line-height: 1.6;
	margin-bottom: 12px;
}

.hero-sub span {
	color: #fbbf24;
	font-weight: 600;
}

.hero-actions {
	margin-top: 4px;
}

.hero-link {
	font-size: 0.85rem;
	color: #e5e7eb;
	text-decoration: none;
	border-bottom: 1px solid transparent;
	opacity: 0.85;
}

.hero-link:hover {
	border-color: #e5e7eb;
	opacity: 1;
}

/* メインエリア */
.main {
	max-width: 960px;
	margin: 0 auto;
	padding: 0 16px 32px;
}

/* ===== グラスっぽいカード ===== */
.card {
	background: rgba(31, 41, 55, 0.92);
	color: #e5e7eb;
	padding: 20px 24px;
	border-radius: 18px;
	border: 1px solid rgba(148, 163, 184, 0.38);
	box-shadow: 0 22px 45px rgba(15, 23, 42, 0.7);
	backdrop-filter: blur(18px);
	-webkit-backdrop-filter: blur(18px);
	margin-bottom: 20px;
	transition: box-shadow 0.2s ease, transform 0.2s ease, border-color 0.2s ease,
		background 0.2s ease;
}

.card:hover {
	transform: translateY(-1px);
	box-shadow: 0 26px 60px rgba(15, 23, 42, 0.85);
	border-color: rgba(249, 250, 251, 0.35);
	background: rgba(31, 41, 55, 0.98);
}

.card-title {
	margin-bottom: 12px;
	font-size: 1.1rem;
	font-weight: 600;
	color: #f3f4f6;
}

/* 入力 */
.label {
	display: block;
	margin: 8px 0 4px;
	font-weight: 500;
	font-size: 0.9rem;
	color: #e5e7eb;
}

.input-wrap {
	position: relative;
}

.input {
	width: 100%;
	padding: 10px 12px;
	border-radius: 999px;
	border: 1px solid rgba(148, 163, 184, 0.55);
	font-size: 0.9rem;
	background: rgba(15, 23, 42, 0.96);
	color: #f9fafb;
	transition: border-color 0.15s ease, box-shadow 0.15s ease,
		background-color 0.15s;
}

.input.small {
	padding: 8px 12px;
	border-radius: 999px;
}

.input::placeholder {
	color: #6b7280;
}

.input:focus {
	outline: none;
	border-color: #f97316;
	background: rgba(15, 23, 42, 1);
	box-shadow: 0 0 0 2px rgba(248, 113, 113, 0.45);
}

.btn-row {
	margin-top: 12px;
	display: flex;
	gap: 8px;
}

/* ヒント */
.hint {
	font-size: 0.8rem;
	color: #9ca3af;
	margin-top: 6px;
}

.highlight {
	background: rgba(248, 113, 113, 0.16);
	color: #fee2e2;
	padding: 2px 8px;
	border-radius: 999px;
}

/* ボタン */
.btn {
	background: linear-gradient(135deg, #f97373, #fb923c);
	color: #111827;
	border: none;
	padding: 10px 18px;
	border-radius: 999px;
	cursor: pointer;
	font-weight: 600;
	font-size: 0.9rem;
	display: inline-flex;
	align-items: center;
	gap: 8px;
	transition: transform 0.12s ease, box-shadow 0.15s ease, opacity 0.15s ease;
	box-shadow: 0 12px 26px rgba(248, 113, 113, 0.55);
}

.btn:hover:not(:disabled) {
	transform: translateY(-1px);
	box-shadow: 0 16px 34px rgba(248, 113, 113, 0.75);
}

.btn:active:not(:disabled) {
	transform: translateY(0);
	box-shadow: 0 8px 20px rgba(248, 113, 113, 0.6);
}

.btn:disabled {
	opacity: 0.7;
	cursor: default;
}

.btn.ghost {
	background: transparent;
	color: #fee2e2;
	border: 1px solid rgba(248, 113, 113, 0.7);
	box-shadow: none;
}

.btn.ghost:hover:not(:disabled) {
	background: rgba(248, 113, 113, 0.12);
}

/* スピナー */
.spinner {
	width: 14px;
	height: 14px;
	border-radius: 999px;
	border: 2px solid rgba(255, 255, 255, 0.6);
	border-top-color: white;
	animation: spin 0.6s linear infinite;
}

.spinner.small {
	width: 12px;
	height: 12px;
}

/* ソート / フィルタ */
.controls {
	display: flex;
	flex-wrap: wrap;
	gap: 12px 16px;
	align-items: flex-end;
	margin-bottom: 8px;
}

.control-group {
	display: flex;
	flex-direction: column;
	gap: 4px;
}

.control-group.wide {
	flex: 1;
}

.control-label {
	font-size: 0.8rem;
	color: #cbd5f5;
}

.select {
	min-width: 160px;
	padding: 8px 10px;
	border-radius: 999px;
	border: 1px solid rgba(148, 163, 184, 0.55);
	background: rgba(15, 23, 42, 0.96);
	color: #e5e7eb;
	font-size: 0.85rem;
}

/* コメントカード */
.card-comments .count {
	font-size: 0.85rem;
	color: #cbd5f5;
	margin-top: 2px;
}

.comment-list {
	list-style: none;
	padding: 0;
	margin: 16px 0;
	display: flex;
	flex-direction: column;
	gap: 12px;
}

.comment-item {
	border-radius: 14px;
	border: 1px solid rgba(148, 163, 184, 0.35);
	padding: 12px;
	background: radial-gradient(
		circle at top left,
		rgba(17, 24, 39, 0.98),
		rgba(15, 23, 42, 0.92)
	);
}

/* transition-group 用 */
.comment-list-enter-active,
.comment-list-leave-active {
	transition: all 0.18s ease-out;
}

.comment-list-enter-from {
	opacity: 0;
	transform: translateY(6px);
}

.comment-list-leave-to {
	opacity: 0;
	transform: translateY(-4px);
}

/* コメント行ヘッダ */
.comment-header {
	display: flex;
	align-items: center;
	gap: 12px;
}

.avatar {
	width: 40px;
	height: 40px;
	border-radius: 999px;
}

.comment-author {
	flex-grow: 1;
}

.author-name {
	font-weight: 600;
	color: #f9fafb;
	text-decoration: none;
}

.author-name:hover {
	text-decoration: underline;
}

.date {
	font-size: 0.75rem;
	color: #9ca3af;
}

.likes {
	font-size: 0.9rem;
	color: #fecaca;
}

.comment-body {
	margin-top: 8px;
	line-height: 1.6;
	font-size: 0.9rem;
	color: #e5e7eb;
}

/* ページネーション */
.pagination {
	display: flex;
	align-items: center;
	justify-content: center;
	gap: 12px;
	margin-top: 4px;
}

.pager-btn {
	padding: 6px 12px;
	font-size: 0.85rem;
	border-radius: 999px;
	border: 1px solid rgba(148, 163, 184, 0.7);
	background: rgba(15, 23, 42, 0.96);
	color: #e5e7eb;
	cursor: pointer;
}

.pager-btn:disabled {
	opacity: 0.4;
	cursor: default;
}

.page-info {
	font-size: 0.85rem;
	color: #e5e7eb;
}

/* 空状態 */
.card-empty {
	text-align: center;
	color: #9ca3af;
}

/* エラー */
.error {
	margin-top: 8px;
	color: #fecaca;
	font-size: 0.85rem;
}

/* 汎用ステータス */
.status {
	font-size: 0.9rem;
	color: #cbd5f5;
}

.status.center {
	text-align: center;
}

/* アニメーション定義 */
@keyframes app-fade-in {
	from {
		opacity: 0;
	}
	to {
		opacity: 1;
	}
}

@keyframes fade-up {
	from {
		opacity: 0;
		transform: translateY(8px);
	}
	to {
		opacity: 1;
	}
}

@keyframes spin {
	to {
		transform: rotate(360deg);
	}
}

/* レスポンシブ */
@media (max-width: 640px) {
	.hero {
		padding-top: 20px;
	}

	.hero-title {
		font-size: 1.6rem;
	}

	.card {
		padding: 16px 14px;
	}

	.comment-item {
		padding: 10px;
	}

	.controls {
		flex-direction: column;
		align-items: stretch;
	}

	.btn-row {
		flex-direction: column;
		align-items: stretch;
	}

	.btn,
	.btn.ghost {
		width: 100%;
		justify-content: center;
	}
}
</style>
