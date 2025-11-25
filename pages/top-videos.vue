<template>
	<div class="app">
		<header class="hero">
			<div class="hero-inner">
				<div class="hero-text">
					<p class="hero-tag">CHANNEL ANALYTICS</p>
					<h1 class="hero-title">コメント数 TOP10 ジェリエル作！</h1>
					<p class="hero-sub">
						任意の YouTube 動画の <span>videoId</span> を入力すると、
						そのチャンネル内で「コメントが多い動画 TOP10」を一覧表示します。
					</p>

					<div class="hero-actions">
						<NuxtLink to="/" class="hero-link">
							コメント一覧ページへ戻る
						</NuxtLink>
					</div>
				</div>
			</div>
		</header>

		<main class="main">
			<!-- 入力カード -->
			<section class="card card-input fade-up">
				<h2 class="card-title">対象動画を指定</h2>

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
						@click="fetchTopVideos"
						:disabled="loading || !videoId"
					>
						<span v-if="!loading">
							この動画のチャンネルのコメント多い動画TOP10取得
						</span>
						<span v-else>
							<span class="spinner"></span> ランキングを計算中...
						</span>
					</button>

					<!-- Clear ボタン -->
					<button
						type="button"
						class="btn ghost"
						@click="clearAll"
						:disabled="loading"
					>
						Clear
					</button>
				</div>

				<p v-if="errorMessage" class="error">
					{{ errorMessage }}
				</p>
			</section>

			<!-- TOP10動画エリア -->
			<section class="card card-list fade-up delay-1">
				<h2 class="card-title">このチャンネルでコメントが多い動画 TOP10</h2>

				<p v-if="loading && !topVideos.length" class="status">
					ランキングを計算中…
				</p>

				<div v-if="topVideos.length" class="controls">
					<div class="control-group">
						<label class="control-label">並び順</label>
						<select v-model="sortKey" class="select">
							<option value="comments_desc">コメント数が多い順</option>
							<option value="views_desc">再生数が多い順</option>
							<option value="date_new">新しい順</option>
							<option value="date_old">古い順</option>
						</select>
					</div>

					<div class="control-group wide">
						<label class="control-label">タイトルフィルタ</label>
						<input
							v-model="filterKeyword"
							class="input small"
							placeholder="例: live / official / 報告"
						/>
					</div>
				</div>

				<p v-if="topVideos.length" class="count">
					全 {{ topVideos.length }} 本中 {{ filteredCount }} 本を表示中
				</p>

				<transition-group
					v-if="displayVideos.length"
					name="video-list"
					tag="ul"
					class="video-list"
				>
					<li
						v-for="(v, idx) in displayVideos"
						:key="v.videoId"
						class="video-item"
					>
						<div class="rank-badge">
							{{ idx + 1 }}
						</div>
						<a
							:href="`https://www.youtube.com/watch?v=${v.videoId}`"
							target="_blank"
							rel="noopener noreferrer"
							class="thumb-link"
						>
							<img
								v-if="v.thumbnailUrl"
								:src="v.thumbnailUrl"
								:alt="v.title"
								class="thumb"
							/>
						</a>
						<div class="video-info">
							<a
								:href="`https://www.youtube.com/watch?v=${v.videoId}`"
								target="_blank"
								rel="noopener noreferrer"
								class="video-title"
							>
								{{ v.title }}
							</a>
							<p class="video-meta">
								コメント数:
								<span class="strong">
									{{ v.commentCount.toLocaleString() }}
								</span>
								／ 再生数:
								<span>{{ v.viewCount.toLocaleString() }}</span>
								／ 公開日:
								<span>{{ formatDate(v.publishedAt) }}</span>
							</p>
						</div>
					</li>
				</transition-group>

				<p
					v-if="topVideos.length && !displayVideos.length && !loading"
					class="status center"
				>
					フィルタ条件に一致する動画がありません。
				</p>

				<p
					v-if="!loading && !topVideos.length && requestedOnce"
					class="status center"
				>
					ランキング対象の動画が見つかりませんでした。
				</p>
			</section>
		</main>
	</div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { NuxtLink } from '#components'

// youtube data APIキー（ローカル用）
const apiKey = 'AIzaSyByrdYL7fducADQbNf_1CVXp2muhroW690'

const videoId = ref('')
const topVideos = ref([])
const loading = ref(false)
const errorMessage = ref('')
const requestedOnce = ref(false)

// ソート / フィルタ
const sortKey = ref('comments_desc') // comments_desc | views_desc | date_new | date_old
const filterKeyword = ref('')

// 共通ヘルパー：YouTube API呼び出し
const fetchJson = async (baseUrl, paramsObj) => {
	const params = new URLSearchParams(paramsObj)
	const url = `${baseUrl}?${params.toString()}`

	const res = await fetch(url)
	if (!res.ok) {
		const text = await res.text()
		throw new Error(`YouTube API error: ${res.status} ${text}`)
	}
	return await res.json()
}

// クリア
const clearAll = () => {
	videoId.value = ''
	topVideos.value = []
	errorMessage.value = ''
	requestedOnce.value = false
	sortKey.value = 'comments_desc'
	filterKeyword.value = ''
}

// チャンネルTOP10取得メイン処理
const fetchTopVideos = async () => {
	if (!videoId.value) {
		errorMessage.value = 'Video ID を入力してください'
		return
	}

	loading.value = true
	errorMessage.value = ''
	requestedOnce.value = true
	topVideos.value = []

	try {
		const vid = videoId.value.trim()
		const maxUploads = 100 // このチャンネルの最新100本からランキング

		// 1. videoId → channelId
		const videoInfo = await fetchJson(
			'https://www.googleapis.com/youtube/v3/videos',
			{
				key: apiKey,
				part: 'snippet',
				id: vid,
			}
		)
		const channelId = videoInfo.items?.[0]?.snippet?.channelId || null
		if (!channelId) {
			throw new Error('channelId を取得できませんでした')
		}

		// 2. channelId → uploads プレイリストID
		const channelInfo = await fetchJson(
			'https://www.googleapis.com/youtube/v3/channels',
			{
				key: apiKey,
				part: 'contentDetails',
				id: channelId,
			}
		)
		const uploadsPlaylistId =
			channelInfo.items?.[0]?.contentDetails?.relatedPlaylists?.uploads
		if (!uploadsPlaylistId) {
			throw new Error('アップロード動画のプレイリストが見つかりません')
		}

		// 3. uploads プレイリスト → videoId を集める（maxUploads まで）
		const videoIds = []
		let nextPageToken = null

		while (videoIds.length < maxUploads) {
			const params = {
				key: apiKey,
				part: 'contentDetails',
				playlistId: uploadsPlaylistId,
				maxResults: '50',
			}
			if (nextPageToken) {
				params.pageToken = nextPageToken
			}

			const playlistData = await fetchJson(
				'https://www.googleapis.com/youtube/v3/playlistItems',
				params
			)

			const ids =
				playlistData.items?.map((item) => item.contentDetails?.videoId) ?? []
			videoIds.push(...ids.filter((id) => typeof id === 'string'))

			nextPageToken = playlistData.nextPageToken
			if (!nextPageToken) break
		}

		const limitedIds = videoIds.slice(0, maxUploads)
		if (!limitedIds.length) {
			throw new Error('このチャンネルの動画が見つかりませんでした')
		}

		// 4. videoIds → videos.list で statistics を取得
		const allVideoItems = []
		const chunkSize = 50

		for (let i = 0; i < limitedIds.length; i += chunkSize) {
			const chunk = limitedIds.slice(i, i + chunkSize)
			const statsData = await fetchJson(
				'https://www.googleapis.com/youtube/v3/videos',
				{
					key: apiKey,
					part: 'snippet,statistics',
					id: chunk.join(','),
				}
			)
			allVideoItems.push(...(statsData.items ?? []))
		}

		// 5. commentCount でソートして TOP10 抜き出し
		const mapped =
			allVideoItems.map((item) => ({
				videoId: item.id,
				title: item.snippet.title,
				thumbnailUrl:
					item.snippet.thumbnails?.medium?.url ||
					item.snippet.thumbnails?.default?.url ||
					'',
				commentCount: Number(item.statistics?.commentCount ?? 0),
				viewCount: Number(item.statistics?.viewCount ?? 0),
				publishedAt: item.snippet.publishedAt,
			})) ?? []

		const top = mapped
			.filter((v) => !Number.isNaN(v.commentCount))
			.sort((a, b) => b.commentCount - a.commentCount)
			.slice(0, 10)

		topVideos.value = top
	} catch (e) {
		console.error(e)
		errorMessage.value = e.message || 'ランキング取得に失敗しました'
	} finally {
		loading.value = false
	}
}

const formatDate = (iso) => {
	const d = new Date(iso)
	if (Number.isNaN(d.getTime())) return iso
	return d.toLocaleString()
}

// ソート＋フィルタ済みTOP10
const displayVideos = computed(() => {
	let arr = topVideos.value.slice()

	// ソート
	if (sortKey.value === 'views_desc') {
		arr.sort((a, b) => (b.viewCount ?? 0) - (a.viewCount ?? 0))
	} else if (sortKey.value === 'date_new') {
		arr.sort(
			(a, b) =>
				new Date(b.publishedAt).getTime() - new Date(a.publishedAt).getTime()
		)
	} else if (sortKey.value === 'date_old') {
		arr.sort(
			(a, b) =>
				new Date(a.publishedAt).getTime() - new Date(b.publishedAt).getTime()
		)
	} else {
		// comments_desc（デフォルト）
		arr.sort((a, b) => (b.commentCount ?? 0) - (a.commentCount ?? 0))
	}

	// フィルタ
	const kw = filterKeyword.value.trim().toLowerCase()
	if (!kw) return arr

	return arr.filter((v) => v.title.toLowerCase().includes(kw))
})

const filteredCount = computed(() => displayVideos.value.length)
</script>

<style scoped>
:global(body) {
	margin: 0;
	background: #050509;
}

:global(*),
:global(*::before),
:global(*::after) {
	box-sizing: border-box;
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

/* 入力フォーム */
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
	box-sizing: border-box;
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

/* Clear / 実行ボタン行 */
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

/* ソート / フィルタ */
.controls {
	display: flex;
	flex-wrap: wrap;
	gap: 12px 16px;
	align-items: flex-end;
	margin-top: 4px;
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
	min-width: 170px;
	padding: 8px 10px;
	border-radius: 999px;
	border: 1px solid rgba(148, 163, 184, 0.55);
	background: rgba(15, 23, 42, 0.96);
	color: #e5e7eb;
	font-size: 0.85rem;
	box-sizing: border-box;
}

/* TOP10 リスト */
.count {
	font-size: 0.85rem;
	color: #cbd5f5;
	margin-top: 8px;
}

.video-list {
	list-style: none;
	padding: 0;
	margin: 16px 0 0;
	display: flex;
	flex-direction: column;
	gap: 12px;
}

.video-item {
	display: flex;
	gap: 12px;
	padding: 10px;
	border-radius: 14px;
	border: 1px solid rgba(148, 163, 184, 0.35);
	background: radial-gradient(
		circle at top left,
		rgba(17, 24, 39, 0.98),
		rgba(15, 23, 42, 0.92)
	);
	align-items: center;
	flex-wrap: wrap;
}

/* transition-group 用 */
.video-list-enter-active,
.video-list-leave-active {
	transition: all 0.18s ease-out;
}

.video-list-enter-from {
	opacity: 0;
	transform: translateY(6px);
}

.video-list-leave-to {
	opacity: 0;
	transform: translateY(-4px);
}

/* ランクバッジ */
.rank-badge {
	width: 28px;
	height: 28px;
	border-radius: 999px;
	background: rgba(248, 113, 113, 0.9);
	color: #111827;
	font-size: 0.9rem;
	font-weight: 700;
	display: flex;
	align-items: center;
	justify-content: center;
	flex-shrink: 0;
}

/* サムネ・タイトル周り */
.thumb-link {
	flex-shrink: 0;
}

.thumb {
	width: 140px;
	height: 78px;
	object-fit: cover;
	border-radius: 10px;
}

.video-info {
	flex: 1;
	min-width: 0;
}

.video-title {
	font-weight: 600;
	text-decoration: none;
	color: #f9fafb;
	word-break: break-word;
	overflow-wrap: anywhere;
}

.video-title:hover {
	text-decoration: underline;
}

.video-meta {
	margin-top: 4px;
	font-size: 0.85rem;
	color: #cbd5f5;
	word-break: break-word;
	overflow-wrap: anywhere;
}

.video-meta .strong {
	color: #fecaca;
	font-weight: 600;
}

/* 汎用ステータス */
.status {
	font-size: 0.9rem;
	color: #cbd5f5;
}

.status.center {
	text-align: center;
}

/* エラー・空状態 */
.error {
	margin-top: 8px;
	color: #fecaca;
	font-size: 0.85rem;
}

.card-empty {
	text-align: center;
	color: #9ca3af;
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
		transform: translateY(0);
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

	.controls {
		flex-direction: column;
		align-items: stretch;
	}

	.video-item {
		padding: 10px;
		flex-direction: column;
		align-items: flex-start;
	}

	.thumb {
		width: 100%;
		height: auto;
	}

	.btn-row {
		flex-direction: column;
	}

	.btn,
	.btn.ghost {
		width: 100%;
		justify-content: center;
	}
}

@media (max-width: 960px) {
	.hero {
		padding: 24px 12px 20px;
	}

	.main {
		padding: 0 12px 28px;
	}

	.card {
		padding: 18px 18px;
	}

	.controls {
		flex-direction: column;
		align-items: stretch;
	}

	.btn-row {
		flex-wrap: wrap;
	}

	.btn,
	.btn.ghost {
		flex: 1;
		justify-content: center;
	}

	.video-item {
		align-items: flex-start;
	}

	.video-info {
		width: 100%;
	}
}

@media (max-width: 768px) {
	.hero {
		text-align: center;
		padding: 24px 12px;
	}

	.hero-actions {
		justify-content: center;
	}

	.card {
		padding: 16px 16px;
	}

	.btn-row {
		flex-direction: column;
	}

	.btn,
	.btn.ghost {
		width: 100%;
	}

	.controls {
		gap: 10px;
	}
}

@media (max-width: 480px) {
	.hero-title {
		font-size: 1.4rem;
	}

	.hero-sub {
		font-size: 0.85rem;
		line-height: 1.45;
	}

	.card-title {
		font-size: 1rem;
	}

	.input,
	.select {
		font-size: 0.85rem;
	}

	.btn-row {
		flex-direction: column;
	}

	.btn,
	.btn.ghost {
		width: 100%;
	}

	.thumb {
		width: 100%;
		height: auto;
	}

	.rank-badge {
		width: 26px;
		height: 26px;
		font-size: 0.8rem;
	}

	.count {
		text-align: left;
	}
}
</style>
