<template>
	<private-view title="Upload & Copy Links">
		<div class="page">
			<section class="intro">
				<p class="intro-copy">
					Upload one or more files, immediately get public links to copy and use in text, buttons, cards, and
					elsewhere
				</p>
			</section>

			<section
				class="drop-zone"
				:class="{ dragging: isDragging, disabled: isUploading || !projectUrl }"
				@dragenter.prevent="isDragging = true"
				@dragover.prevent="isDragging = true"
				@dragleave.prevent="leaveDropZone"
				@drop.prevent="handleDrop"
			>
				<input
					ref="fileInput"
					class="file-input"
					type="file"
					multiple
					:disabled="isUploading || !projectUrl"
					@change="handleSelection"
				/>
				<div class="upload-mark">
					<v-icon name="upload" />
				</div>
				<div>
					<h2>{{ dropZoneTitle }}</h2>
					<p>{{ isUploading ? uploadProgress : 'or choose them from your device' }}</p>
				</div>
				<v-button :disabled="isUploading || !projectUrl" @click="openFilePicker">
					{{ isUploading ? 'Uploading' : 'Choose files' }}
				</v-button>
			</section>

			<div class="permission-note" :class="{ warning: !projectUrl }">
				<v-icon :name="projectUrl ? 'link' : 'warning'" />
				<span v-if="projectUrl">Links use the configured Project URL: <strong>{{ projectUrl }}</strong></span>
				<span v-else>Set a <strong>Project URL</strong> in Directus Project Settings before uploading.</span>
			</div>

			<section v-if="uploads.length" class="results">
				<header class="results-header">
					<div>
						<p class="eyebrow">Recent uploads</p>
						<h2>{{ successfulUploads.length }} {{ successfulUploads.length === 1 ? 'link' : 'links' }} ready</h2>
					</div>
					<div class="result-actions">
						<v-button v-if="successfulUploads.length" secondary @click="copyAll">
							<v-icon name="content_copy" left />
							Copy all
						</v-button>
						<v-button secondary @click="clearUploads">Clear</v-button>
					</div>
				</header>

				<ul class="file-list">
					<li v-for="upload in uploads" :key="upload.key" class="file-row">
						<div class="file-icon" :class="upload.status">
							<v-progress-circular v-if="upload.status === 'uploading'" indeterminate small />
							<v-icon v-else :name="upload.status === 'done' ? 'insert_drive_file' : 'error'" />
						</div>
						<div class="file-details">
							<div class="file-heading">
								<strong>{{ upload.name }}</strong>
								<span>{{ formatBytes(upload.size) }}</span>
							</div>
							<a v-if="upload.url" :href="upload.url" target="_blank" rel="noopener noreferrer">
								{{ upload.url }}
							</a>
							<p v-else-if="upload.error" class="error-message">{{ upload.error }}</p>
							<p v-else class="uploading-message">Uploading...</p>
						</div>
						<button
							v-if="upload.url"
							class="copy-button"
							type="button"
							:aria-label="`Copy link for ${upload.name}`"
							@click="copyLink(upload)"
						>
							<v-icon :name="copiedKey === upload.key ? 'check' : 'content_copy'" />
							<span>{{ copiedKey === upload.key ? 'Copied' : 'Copy' }}</span>
						</button>
					</li>
				</ul>
			</section>
		</div>
	</private-view>
</template>

<script setup>
import { defineStore } from 'pinia';
import { computed, onBeforeUnmount, onMounted, ref } from 'vue';
import { useApi, useStores } from '@directus/extensions-sdk';

const api = useApi();
const { useSettingsStore } = useStores();
const settingsStore = useSettingsStore();
const useNavBarStore = defineStore('nav-bar-store', {});
let navBarWasCollapsed;
const fileInput = ref(null);
const uploads = ref([]);
const isDragging = ref(false);
const activeUploads = ref(0);
const completedInBatch = ref(0);
const batchSize = ref(0);
const copiedKey = ref(null);

const isUploading = computed(() => activeUploads.value > 0);
const successfulUploads = computed(() => uploads.value.filter((upload) => upload.status === 'done'));
const uploadProgress = computed(() => `${completedInBatch.value} of ${batchSize.value} complete`);
const projectUrl = computed(() => settingsStore.settings?.project_url?.trim() || '');
const dropZoneTitle = computed(() => {
	if (!projectUrl.value) return 'Project URL required';
	return isUploading.value ? 'Uploading your files...' : 'Drop files here';
});

onMounted(() => {
	const navBarStore = useNavBarStore();
	if (typeof navBarStore.collapse !== 'function') return;

	navBarWasCollapsed = navBarStore.collapsed;
	navBarStore.collapse();
});

onBeforeUnmount(() => {
	const navBarStore = useNavBarStore();
	if (navBarWasCollapsed === false && typeof navBarStore.expand === 'function') navBarStore.expand();
});

function openFilePicker() {
	fileInput.value?.click();
}

function handleSelection(event) {
	queueFiles(event.target.files);
	event.target.value = '';
}

function handleDrop(event) {
	isDragging.value = false;
	if (!isUploading.value && projectUrl.value) queueFiles(event.dataTransfer.files);
}

function leaveDropZone(event) {
	if (!event.currentTarget.contains(event.relatedTarget)) isDragging.value = false;
}

async function queueFiles(fileList) {
	const files = Array.from(fileList ?? []);
	if (!files.length || isUploading.value || !projectUrl.value) return;

	batchSize.value = files.length;
	completedInBatch.value = 0;
	activeUploads.value = files.length;

	const newUploads = files.map((file) => ({
		key: `${Date.now()}-${crypto.randomUUID()}`,
		name: file.name,
		size: file.size,
		status: 'uploading',
		url: null,
		error: null,
		file,
	}));

	uploads.value.unshift(...newUploads);
	await Promise.allSettled(newUploads.map(uploadFile));
}

async function uploadFile(upload) {
	const formData = new FormData();
	formData.append('file', upload.file, upload.name);
	const trackedUpload = uploads.value.find((item) => item.key === upload.key);

	try {
		const response = await api.post('/files', formData);
		const uploadedFile = response.data.data;
		trackedUpload.url = createDownloadUrl(uploadedFile.id);
		trackedUpload.status = 'done';
	} catch (error) {
		trackedUpload.status = 'error';
		trackedUpload.error = error.response?.data?.errors?.[0]?.message || error.message || 'Upload failed';
	} finally {
		delete trackedUpload.file;
		activeUploads.value -= 1;
		completedInBatch.value += 1;
	}
}

function createDownloadUrl(id) {
	const root = projectUrl.value.endsWith('/') ? projectUrl.value : `${projectUrl.value}/`;
	return new URL(`downloads/${id}`, root).href;
}

async function copyLink(upload) {
	await copyText(upload.url);
	copiedKey.value = upload.key;
	window.setTimeout(() => {
		if (copiedKey.value === upload.key) copiedKey.value = null;
	}, 1600);
}

async function copyAll() {
	await copyText(successfulUploads.value.map((upload) => upload.url).join('\n'));
	copiedKey.value = 'all';
	window.setTimeout(() => {
		if (copiedKey.value === 'all') copiedKey.value = null;
	}, 1600);
}

async function copyText(text) {
	if (navigator.clipboard?.writeText) {
		await navigator.clipboard.writeText(text);
		return;
	}

	const textArea = document.createElement('textarea');
	textArea.value = text;
	textArea.style.position = 'fixed';
	textArea.style.opacity = '0';
	document.body.appendChild(textArea);
	textArea.select();
	document.execCommand('copy');
	textArea.remove();
}

function clearUploads() {
	if (!isUploading.value) uploads.value = [];
}

function formatBytes(bytes) {
	if (bytes === 0) return '0 B';
	const units = ['B', 'KB', 'MB', 'GB', 'TB'];
	const index = Math.min(Math.floor(Math.log(bytes) / Math.log(1024)), units.length - 1);
	const value = bytes / 1024 ** index;
	return `${value >= 10 || index === 0 ? value.toFixed(0) : value.toFixed(1)} ${units[index]}`;
}
</script>

<style scoped>
.page {
	--module-ink: var(--theme--foreground, #17212b);
	--module-muted: var(--theme--foreground-subdued, #667085);
	--module-surface: var(--theme--background-normal, #fff);
	--module-border: var(--theme--border-color, #d3dae4);
	max-width: 1040px;
	margin: 0 auto;
	padding: 48px 32px 80px;
}

.intro {
	max-width: 720px;
	margin-bottom: 34px;
}

.eyebrow {
	margin: 0 0 8px;
	color: var(--theme--primary, #6644ff);
	font-size: 12px;
	font-weight: 700;
	letter-spacing: 0.12em;
	text-transform: uppercase;
}

.intro-copy {
	max-width: 650px;
	margin: 0;
	color: var(--module-muted);
	font-size: 17px;
	line-height: 1.65;
}

.drop-zone {
	position: relative;
	display: grid;
	grid-template-columns: auto 1fr auto;
	gap: 22px;
	align-items: center;
	min-height: 190px;
	padding: 36px;
	overflow: hidden;
	background:
		linear-gradient(135deg, color-mix(in srgb, var(--theme--primary, #6644ff) 8%, transparent), transparent 48%),
		var(--module-surface);
	border: 2px dashed var(--module-border);
	border-radius: 18px;
	transition: border-color 160ms ease, transform 160ms ease, background-color 160ms ease;
}

.drop-zone.dragging {
	background-color: color-mix(in srgb, var(--theme--primary, #6644ff) 7%, var(--module-surface));
	border-color: var(--theme--primary, #6644ff);
	transform: translateY(-2px);
}

.drop-zone.disabled {
	cursor: wait;
}

.file-input {
	position: absolute;
	width: 1px;
	height: 1px;
	opacity: 0;
	pointer-events: none;
}

.upload-mark {
	display: grid;
	width: 68px;
	height: 68px;
	place-items: center;
	color: var(--theme--primary, #6644ff);
	background: color-mix(in srgb, var(--theme--primary, #6644ff) 13%, transparent);
	border-radius: 16px;
}

.upload-mark :deep(.v-icon) {
	font-size: 32px;
}

.drop-zone h2,
.results h2 {
	margin: 0;
	color: var(--module-ink);
	font-size: 22px;
	font-weight: 700;
}

.drop-zone p {
	margin: 7px 0 0;
	color: var(--module-muted);
}

.permission-note {
	display: flex;
	gap: 9px;
	align-items: center;
	margin: 14px 4px 0;
	color: var(--module-muted);
	font-size: 13px;
}

.permission-note :deep(.v-icon) {
	color: var(--theme--primary, #6644ff);
	font-size: 17px;
}

.permission-note.warning,
.permission-note.warning :deep(.v-icon) {
	color: var(--theme--warning, #f2c94c);
}

.results {
	margin-top: 52px;
}

.results-header {
	display: flex;
	justify-content: space-between;
	gap: 24px;
	align-items: flex-end;
	margin-bottom: 18px;
}

.result-actions {
	display: flex;
	gap: 10px;
}

.file-list {
	padding: 0;
	margin: 0;
	overflow: hidden;
	background: var(--module-surface);
	border: 1px solid var(--module-border);
	border-radius: 14px;
	list-style: none;
}

.file-row {
	display: grid;
	grid-template-columns: auto minmax(0, 1fr) auto;
	gap: 16px;
	align-items: center;
	padding: 18px 20px;
}

.file-row + .file-row {
	border-top: 1px solid var(--module-border);
}

.file-icon {
	display: grid;
	width: 42px;
	height: 42px;
	place-items: center;
	color: var(--theme--primary, #6644ff);
	background: color-mix(in srgb, var(--theme--primary, #6644ff) 10%, transparent);
	border-radius: 10px;
}

.file-icon.error {
	color: var(--theme--danger, #e35169);
	background: color-mix(in srgb, var(--theme--danger, #e35169) 10%, transparent);
}

.file-details {
	min-width: 0;
}

.file-heading {
	display: flex;
	gap: 12px;
	align-items: baseline;
	min-width: 0;
}

.file-heading strong {
	overflow: hidden;
	color: var(--module-ink);
	text-overflow: ellipsis;
	white-space: nowrap;
}

.file-heading span {
	flex: none;
	color: var(--module-muted);
	font-size: 12px;
}

.file-details a,
.file-details p {
	display: block;
	margin: 5px 0 0;
	overflow: hidden;
	font-size: 13px;
	text-overflow: ellipsis;
	white-space: nowrap;
}

.file-details a {
	color: var(--theme--primary, #6644ff);
}

.error-message {
	color: var(--theme--danger, #e35169);
}

.uploading-message {
	color: var(--module-muted);
}

.copy-button {
	display: flex;
	gap: 7px;
	align-items: center;
	padding: 9px 11px;
	color: var(--module-muted);
	font: inherit;
	font-size: 13px;
	font-weight: 600;
	background: transparent;
	border: 0;
	border-radius: 8px;
	cursor: pointer;
}

.copy-button:hover {
	color: var(--theme--primary, #6644ff);
	background: color-mix(in srgb, var(--theme--primary, #6644ff) 8%, transparent);
}

.copy-button :deep(.v-icon) {
	font-size: 18px;
}

@media (max-width: 700px) {
	.page {
		padding: 30px 18px 60px;
	}

	.drop-zone {
		grid-template-columns: 1fr;
		justify-items: center;
		padding: 30px 22px;
		text-align: center;
	}

	.results-header {
		align-items: flex-start;
		flex-direction: column;
	}

	.result-actions {
		width: 100%;
	}

	.file-row {
		grid-template-columns: auto minmax(0, 1fr);
		padding: 16px;
	}

	.copy-button {
		grid-column: 2;
		justify-self: start;
		padding-left: 0;
	}

	.permission-note {
		align-items: flex-start;
	}
}
</style>
