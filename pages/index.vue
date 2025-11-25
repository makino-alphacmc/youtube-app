<template>
	<div class="app">
		<header class="hero">
			<div class="hero-inner">
				<p class="hero-tag">CHANNEL ANALYTICS SUITE</p>
				<h1 class="hero-title">YouTube ダッシュボード</h1>
				<p class="hero-sub">
					視聴者コメントの分析、ライブチャットの追跡、チャンネル内ランキングなど、
					3つのビューから瞬時にアクセスできます。
				</p>
			</div>
		</header>

		<main class="main">
			<section class="menu-panel">
				<ul class="menu-list">
					<li v-for="item in menuLinks" :key="item.to">
						<NuxtLink :to="item.to" class="menu-card">
							<div class="menu-card-icon">{{ item.icon }}</div>
							<div class="menu-card-body">
								<p class="menu-card-label">{{ item.label }}</p>
								<p class="menu-card-desc">{{ item.desc }}</p>
							</div>
							<span class="menu-card-arrow">→</span>
						</NuxtLink>
					</li>
				</ul>
			</section>
		</main>
	</div>
</template>

<script setup>
const menuLinks = [
	{
		label: 'コメント一覧ビュー',
		desc: '任意動画のコメントを取得し、ソート・フィルタ・ページングで分析できます。',
		to: '/comments',
		icon: '💬',
	},
	{
		label: 'ライブチャットビュー',
		desc: 'ライブ配信中のチャットをタイムライン付きで取得、追加読み込みにも対応。',
		to: '/live-chat',
		icon: '📺',
	},
	{
		label: 'コメント数 TOP10',
		desc: 'チャンネル内でコメントが多い動画をランキング形式で表示します。',
		to: '/top-videos',
		icon: '🏆',
	},
]
</script>

<style scoped>
::global(body) {
	margin: 0;
	background: #050509;
	font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI',
		sans-serif;
	color: #e5e7eb;
}

::global(*),
::global(*::before),
::global(*::after) {
	box-sizing: border-box;
}

.app {
	min-height: 100vh;
	background: radial-gradient(
			circle at top left,
			#3b3f46 0,
			#181b20 40%,
			#050608 75%
		)
		fixed;
	padding: 32px 16px 64px;
}

.hero {
	padding: 32px 0 12px;
}

.hero-inner {
	max-width: 720px;
	margin: 0 auto;
	text-align: center;
}

.hero-tag {
	font-size: 0.75rem;
	letter-spacing: 0.16em;
	color: #9ca3af;
	margin-bottom: 6px;
	text-transform: uppercase;
}

.hero-title {
	font-size: 2.2rem;
	margin: 0 0 12px;
	color: #f9fafb;
}

.hero-sub {
	color: #d1d5db;
	line-height: 1.8;
	margin: 0;
}

.main {
	max-width: 900px;
	margin: 0 auto;
	padding: 24px 0;
}

.menu-panel {
	background: rgba(15, 23, 42, 0.9);
	border: 1px solid rgba(148, 163, 184, 0.35);
	border-radius: 20px;
	padding: 28px 32px;
	box-shadow: 0 20px 50px rgba(15, 23, 42, 0.7);
	backdrop-filter: blur(18px);
}

.menu-title {
	margin: 0 0 20px;
	font-size: 1.2rem;
	font-weight: 600;
}

.menu-list {
	list-style: none;
	padding: 0;
	margin: 0;
	display: flex;
	flex-direction: column;
	gap: 16px;
}

.menu-card {
	display: flex;
	align-items: center;
	gap: 16px;
	padding: 16px 18px;
	border-radius: 16px;
	text-decoration: none;
	border: 1px solid rgba(148, 163, 184, 0.25);
	background: rgba(30, 41, 59, 0.85);
	transition: transform 0.18s ease, box-shadow 0.18s ease,
		border-color 0.18s ease;
	color: inherit;
}

.menu-card:hover {
	transform: translateY(-3px);
	border-color: rgba(248, 113, 113, 0.8);
	box-shadow: 0 18px 40px rgba(248, 113, 113, 0.25);
}

.menu-card-icon {
	width: 52px;
	height: 52px;
	border-radius: 14px;
	background: rgba(248, 113, 113, 0.15);
	display: flex;
	align-items: center;
	justify-content: center;
	font-size: 1.5rem;
}

.menu-card-body {
	flex: 1;
	min-width: 0;
}

.menu-card-label {
	font-size: 1.05rem;
	font-weight: 600;
	margin: 0;
	color: #f3f4f6;
}

.menu-card-desc {
	margin: 4px 0 0;
	color: #9ca3af;
	font-size: 0.9rem;
	line-height: 1.6;
}

.menu-card-arrow {
	font-size: 1.2rem;
	color: #fbbf24;
}

@media (max-width: 640px) {
	.menu-panel {
		padding: 20px 18px;
	}

	.menu-card {
		align-items: flex-start;
	}

	.menu-card-icon {
		width: 46px;
		height: 46px;
	}
}
</style>
