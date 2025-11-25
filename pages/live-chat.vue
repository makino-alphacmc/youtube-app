<template>
	<div class="app">
		<MainMenu />
		<header class="hero">
			<div class="hero-inner">
				<div class="hero-text">
					<p class="hero-tag">CHANNEL ANALYTICS</p>
					<h1 class="hero-title">ライブチャット ビューア ジェリエル作！</h1>
					<p class="hero-sub">
						ライブ配信中の YouTube 動画の <span>videoId</span> を入力すると、
						その配信のライブチャットを
						<span>再生時間（オフセット）付き</span>で表示します。
					</p>

					<div class="hero-actions">
						<NuxtLink to="/" class="hero-link"> コメント一覧ページへ </NuxtLink>
						<NuxtLink to="/top-videos" class="hero-link second">
							コメント数 TOP10 ページへ
						</NuxtLink>
					</div>
				</div>
			</div>
		</header>

		<main class="main">
			<!-- 入力カード -->
			<section class="card card-input fade-up">
				<h2 class="card-title">ライブ配信動画を指定</h2>

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
						@click="startFetch"
						:disabled="loading || !videoId"
					>
						<span v-if="!loading"> ライブチャットを取得（現在まで） </span>
						<span v-else> <span class="spinner"></span> 取得中... </span>
					</button>

					<button
						type="button"
						class="btn ghost"
						@click="clearAll"
						:disabled="loading && !videoId"
					>
						Clear
					</button>
				</div>

				<p v-if="errorMessage" class="error">
					{{ errorMessage }}
				</p>
			</section>

			<!-- チャット一覧 -->
			<section class="card card-list fade-up delay-1">
				<h2 class="card-title">ライブチャット一覧</h2>

				<p v-if="loading && !messages.length" class="status">
					ライブチャットを取得中…
				</p>

				<p v-if="hasLiveChatId" class="status small">
					liveChatId: <code>{{ liveChatId }}</code>
				</p>

				<p v-if="messages.length" class="count">
					現在 {{ messages.length }} 件のチャットメッセージを取得済み
				</p>

				<transition-group
					v-if="messages.length"
					name="chat-list"
					tag="ul"
					class="chat-list"
				>
					<li v-for="m in messages" :key="m.id" class="chat-item">
						<div class="chat-header">
							<div class="offset-badge">
								{{ m.offsetLabel }}
							</div>

							<img
								v-if="m.authorProfileImageUrl"
								:src="m.authorProfileImageUrl"
								class="avatar"
							/>

							<div class="chat-author">
								<span class="author-name">{{ m.author }}</span>
								<span class="date">{{ formatDate(m.publishedAt) }}</span>
							</div>
						</div>

						<p class="chat-body" v-html="m.message" />
					</li>
				</transition-group>

				<p
					v-if="!loading && !messages.length && requestedOnce"
					class="status center"
				>
					チャットメッセージが取得できませんでした。<br />
					・動画がライブ配信中か<br />
					・ライブチャットが有効か<br />
					を確認してください。
				</p>

				<!-- 追加読み込み -->
				<div class="more-wrap" v-if="nextPageToken && messages.length">
					<button @click="loadMore" class="btn ghost" :disabled="loading">
						<span v-if="!loading">さらに過去のチャットを読み込む</span>
						<span v-else>
							<span class="spinner small"></span> 読み込み中...
						</span>
					</button>
				</div>
			</section>
		</main>
	</div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { NuxtLink } from '#components'

const runtimeConfig = useRuntimeConfig()
const apiKey = runtimeConfig.public.youtubeApiKey || ''

const videoId = ref('')
const liveChatId = ref('')
const messages = ref([])
const nextPageToken = ref(null)
const loading = ref(false)
const errorMessage = ref('')
const requestedOnce = ref(false)

const hasLiveChatId = computed(() => !!liveChatId.value)

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

// videoId → activeLiveChatId を取得
const resolveLiveChatId = async (vid) => {
	if (!apiKey) {
		throw new Error('環境変数 YOUTUBE_API_KEY が設定されていません')
	}
	const data = await fetchJson('https://www.googleapis.com/youtube/v3/videos', {
		key: apiKey,
		part: 'liveStreamingDetails',
		id: vid,
	})

	const details = data.items?.[0]?.liveStreamingDetails
	const activeId = details?.activeLiveChatId || null
	if (!activeId) {
		throw new Error(
			'この動画の activeLiveChatId を取得できませんでした（ライブ配信中ではない可能性があります）'
		)
	}
	return activeId
}

// liveChatMessages を取得
const callLiveChatApi = async (options = {}) => {
	if (!apiKey) {
		throw new Error('環境変数 YOUTUBE_API_KEY が設定されていません')
	}
	const params = {
		key: apiKey,
		part: 'snippet,authorDetails',
		liveChatId: options.liveChatId,
		maxResults: '200',
	}
	if (options.pageToken) params.pageToken = options.pageToken

	const data = await fetchJson(
		'https://www.googleapis.com/youtube/v3/liveChat/messages',
		params
	)

	const mapped =
		data.items?.map((item) => {
			const s = item.snippet
			const a = item.authorDetails
			const offsetMs = Number(s.videoOffsetTimeMs ?? 0)
			const offsetSec = Math.floor(offsetMs / 1000)

			return {
				id: item.id,
				author: a?.displayName ?? 'Unknown',
				authorProfileImageUrl: a?.profileImageUrl ?? null,
				message: s.displayMessage ?? '',
				publishedAt: s.publishedAt,
				offsetSeconds: offsetSec,
				offsetLabel: formatOffset(offsetSec),
			}
		}) ?? []

	return {
		messages: mapped,
		nextPageToken: data.nextPageToken ?? null,
	}
}

const startFetch = async () => {
	if (!videoId.value) {
		errorMessage.value = 'Video ID を入力してください'
		return
	}

	loading.value = true
	errorMessage.value = ''
	requestedOnce.value = true
	messages.value = []
	liveChatId.value = ''
	nextPageToken.value = null

	try {
		const vid = videoId.value.trim()

		// 1. videoId → liveChatId
		const activeId = await resolveLiveChatId(vid)
		liveChatId.value = activeId

		// 2. liveChatMessages を一気に取得（現時点まで）
		const result = await callLiveChatApi({ liveChatId: activeId })
		messages.value = result.messages
		nextPageToken.value = result.nextPageToken
	} catch (e) {
		console.error(e)
		errorMessage.value = e.message || 'ライブチャット取得に失敗しました'
	} finally {
		loading.value = false
	}
}

const loadMore = async () => {
	if (!liveChatId.value || !nextPageToken.value) return

	loading.value = true
	errorMessage.value = ''

	try {
		const result = await callLiveChatApi({
			liveChatId: liveChatId.value,
			pageToken: nextPageToken.value,
		})
		messages.value.push(...result.messages)
		nextPageToken.value = result.nextPageToken
	} catch (e) {
		console.error(e)
		errorMessage.value = e.message || '追加読み込みに失敗しました'
	} finally {
		loading.value = false
	}
}

const clearAll = () => {
	videoId.value = ''
	liveChatId.value = ''
	messages.value = []
	nextPageToken.value = null
	errorMessage.value = ''
	requestedOnce.value = false
}

const formatDate = (iso) => {
	const d = new Date(iso)
	return Number.isNaN(d.getTime()) ? iso : d.toLocaleString()
}

const formatOffset = (seconds) => {
	if (!seconds || seconds < 0) return '0:00'
	const h = Math.floor(seconds / 3600)
	const m = Math.floor((seconds % 3600) / 60)
	const s = seconds % 60

	const mm = String(m).padStart(1, '0')
	const ss = String(s).padStart(2, '0')

	if (h > 0) {
		const hh = String(h)
		const mm2 = String(m).padStart(2, '0')
		return `${hh}:${mm2}:${ss}`
	}
	return `${mm}:${ss}`
}
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

/* ヒーロー */
.hero {
	padding: 32px 16px 24px;
}

.hero-inner {
	max-width: 960px;
	margin: 0 auto;
}

.hero-text {
	max-width: 620px;
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
	margin-top: 8px;
	display: flex;
	flex-wrap: wrap;
	gap: 12px;
}

.hero-link {
	font-size: 0.85rem;
	color: #e5e7eb;
	text-decoration: none;
	border-bottom: 1px solid transparent;
	opacity: 0.85;
}

.hero-link.second {
	opacity: 0.7;
	font-size: 0.8rem;
}

.hero-link:hover {
	border-color: #e5e7eb;
	opacity: 1;
}

/* メイン */
.main {
	max-width: 960px;
	margin: 0 auto;
	padding: 0 16px 32px;
}

/* カード */
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
.btn-row {
	margin-top: 14px;
	display: flex;
	gap: 10px;
	flex-wrap: wrap;
}

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

/* チャットリスト */
.chat-list {
	list-style: none;
	padding: 0;
	margin: 16px 0 0;
	display: flex;
	flex-direction: column;
	gap: 12px;
}

.chat-item {
	border-radius: 14px;
	border: 1px solid rgba(148, 163, 184, 0.35);
	padding: 10px 12px;
	background: radial-gradient(
		circle at top left,
		rgba(17, 24, 39, 0.98),
		rgba(15, 23, 42, 0.92)
	);
}

/* transition-group 用 */
.chat-list-enter-active,
.chat-list-leave-active {
	transition: all 0.18s ease-out;
}

.chat-list-enter-from {
	opacity: 0;
	transform: translateY(6px);
}

.chat-list-leave-to {
	opacity: 0;
	transform: translateY(-4px);
}

/* チャット行ヘッダ */
.chat-header {
	display: flex;
	align-items: center;
	gap: 10px;
	flex-wrap: wrap;
}

.offset-badge {
	min-width: 54px;
	padding: 3px 8px;
	border-radius: 999px;
	font-size: 0.75rem;
	font-weight: 600;
	text-align: center;
	background: rgba(55, 65, 81, 0.95);
	color: #e5e7eb;
}

.avatar {
	width: 32px;
	height: 32px;
	border-radius: 999px;
}

.chat-author {
	flex: 1;
	min-width: 0;
}

.author-name {
	font-weight: 600;
	color: #f9fafb;
	display: block;
	font-size: 0.9rem;
}

.date {
	font-size: 0.75rem;
	color: #9ca3af;
}

.chat-body {
	margin-top: 6px;
	font-size: 0.9rem;
	line-height: 1.6;
	color: #e5e7eb;
	word-break: break-word;
	overflow-wrap: anywhere;
}

/* 下部ボタン */
.more-wrap {
	margin-top: 12px;
	text-align: center;
}

/* ステータス・エラー */
.status {
	font-size: 0.9rem;
	color: #cbd5f5;
}

.status.small {
	font-size: 0.75rem;
	margin-top: 6px;
	opacity: 0.85;
}

.status.center {
	text-align: center;
}

.error {
	margin-top: 8px;
	color: #fecaca;
	font-size: 0.85rem;
}

/* カウント */
.count {
	font-size: 0.85rem;
	color: #cbd5f5;
	margin-top: 8px;
}

/* アニメーション */
@keyframes app-fade-in {
	from {
		opacity: 0;
	}
	to {
		opacity: 1;
	}
}

.fade-up {
	opacity: 0;
	transform: translateY(8px);
	animation: fade-up 0.4s ease-out forwards;
}

.delay-1 {
	animation-delay: 0.08s;
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

	.chat-item {
		padding: 8px 10px;
	}

	.hero-actions {
		flex-direction: column;
		align-items: flex-start;
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

	.btn-row {
		flex-direction: column;
		align-items: stretch;
	}

	.btn,
	.btn.ghost {
		width: 100%;
		justify-content: center;
	}

	.chat-header {
		align-items: flex-start;
	}

	.offset-badge {
		order: -1;
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
		gap: 8px;
	}

	.status,
	.count {
		text-align: center;
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

	.chat-item {
		padding: 8px 10px;
	}

	.chat-header {
		gap: 8px;
	}

	.status {
		font-size: 0.8rem;
	}

	.count {
		text-align: left;
	}
}
</style>
