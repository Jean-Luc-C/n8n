<template>
	<div :class="$style.block">
		<header :class="$style.blockHeader" @click="onBlockHeaderClick">
			<button :class="$style.blockToggle">
				<font-awesome-icon :icon="isExpanded ? 'angle-down' : 'angle-up'" size="lg" />
			</button>
			<p :class="$style.blockTitle">{{ capitalize(runData.inOut) }}</p>
			<!-- @click.stop to prevent event from bubbling to blockHeader and toggling expanded state when clicking on rawSwitch -->
			<el-switch
				v-if="contentParsed"
				@click.stop
				:class="$style.rawSwitch"
				active-text="RAW JSON"
				v-model="isShowRaw"
			/>
		</header>
		<main
			:class="{
				[$style.blockContent]: true,
				[$style.blockContentExpanded]: isExpanded,
			}"
		>
			<div
				:key="index"
				v-for="({ parsedContent, raw }, index) in parsedRun"
				:class="$style.contentText"
				:data-content-type="parsedContent?.type"
			>
				<template v-if="parsedContent && !isShowRaw">
					<template v-if="parsedContent.type === 'json'">
						<vue-markdown
							:source="jsonToMarkdown(parsedContent.data as JsonMarkdown)"
							:class="$style.markdown"
						/>
					</template>
					<template v-if="parsedContent.type === 'markdown'">
						<vue-markdown :source="parsedContent.data" :class="$style.markdown" />
					</template>
					<p
						:class="$style.runText"
						v-if="parsedContent.type === 'text'"
						v-text="parsedContent.data"
					/>
				</template>
				<!-- We weren't able to parse text or raw switch -->
				<template v-else>
					<div :class="$style.rawContent">
						<n8n-icon-button
							size="small"
							:class="$style.copyToClipboard"
							type="secondary"
							@click="copyToClipboard(raw)"
							:title="$locale.baseText('nodeErrorView.copyToClipboard')"
							icon="copy"
						/>
						<vue-markdown :source="jsonToMarkdown(raw as JsonMarkdown)" :class="$style.markdown" />
					</div>
				</template>
			</div>
		</main>
	</div>
</template>

<script lang="ts" setup>
import type { IAiDataContent } from '@/Interface';
import { capitalize } from 'lodash-es';
import { ref, onMounted } from 'vue';
import type { ParsedAiContent } from './useAiContentParsers';
import { useAiContentParsers } from './useAiContentParsers';
import VueMarkdown from 'vue-markdown-render';
import { useCopyToClipboard, useI18n, useToast } from '@/composables';
import { NodeConnectionType, type IDataObject } from 'n8n-workflow';

const props = defineProps<{
	runData: IAiDataContent;
}>();

const i18n = useI18n();
const contentParsers = useAiContentParsers();
// eslint-disable-next-line @typescript-eslint/no-use-before-define
const isExpanded = ref(getInitialExpandedState());
const isShowRaw = ref(false);
const contentParsed = ref(false);
const parsedRun = ref(undefined as ParsedAiContent | undefined);

function getInitialExpandedState() {
	const collapsedTypes = {
		input: [NodeConnectionType.AiDocument, NodeConnectionType.AiTextSplitter],
		output: [
			NodeConnectionType.AiDocument,
			NodeConnectionType.AiEmbedding,
			NodeConnectionType.AiTextSplitter,
			NodeConnectionType.AiVectorStore,
		],
	};

	return !collapsedTypes[props.runData.inOut].includes(props.runData.type);
}

function parseAiRunData(run: IAiDataContent) {
	if (!run.data) {
		return;
	}
	const parsedData = contentParsers.parseAiRunData(run.data, run.type);

	return parsedData;
}

function isMarkdown(content: JsonMarkdown): boolean {
	if (typeof content !== 'string') return false;
	const markdownPatterns = [
		/^# .+/gm, // headers
		/\*{1,2}.+\*{1,2}/g, // emphasis and strong
		/\[.+\]\(.+\)/g, // links
		/```[\s\S]+```/g, // code blocks
	];

	return markdownPatterns.some((pattern) => pattern.test(content));
}

function formatToJsonMarkdown(data: string): string {
	return '```json\n' + data + '\n```';
}

type JsonMarkdown = string | object | Array<string | object>;

function jsonToMarkdown(data: JsonMarkdown): string {
	if (isMarkdown(data)) return data as string;

	if (Array.isArray(data) && data.length && typeof data[0] !== 'number') {
		const markdownArray = data.map((item: JsonMarkdown) => jsonToMarkdown(item));

		return markdownArray.join('\n\n').trim();
	}

	if (typeof data === 'string') {
		return formatToJsonMarkdown(data);
	}

	return formatToJsonMarkdown(JSON.stringify(data, null, 2));
}

function setContentParsed(content: ParsedAiContent): void {
	contentParsed.value = !!content.find((item) => {
		if (item.parsedContent?.parsed === true) {
			return true;
		}
		return false;
	});
}

function onBlockHeaderClick() {
	isExpanded.value = !isExpanded.value;
}

function copyToClipboard(content: IDataObject | IDataObject[]) {
	const copyToClipboardFn = useCopyToClipboard();
	const { showMessage } = useToast();

	try {
		copyToClipboardFn(JSON.stringify(content, undefined, 2));
		showMessage({
			title: i18n.baseText('generic.copiedToClipboard'),
			type: 'success',
		});
	} catch (err) {}
}

onMounted(() => {
	parsedRun.value = parseAiRunData(props.runData);
	if (parsedRun.value) {
		setContentParsed(parsedRun.value);
	}
});
</script>

<style lang="scss" module>
.copyToClipboard {
	position: absolute;
	right: var(--spacing-s);
	top: var(--spacing-s);
}
.rawContent {
	position: relative;
}
.markdown {
	& {
		white-space: pre-wrap;

		h1 {
			font-size: var(--font-size-xl);
			line-height: var(--font-line-height-xloose);
		}

		h2 {
			font-size: var(--font-size-l);
			line-height: var(--font-line-height-loose);
		}

		h3 {
			font-size: var(--font-size-m);
			line-height: var(--font-line-height-regular);
		}

		pre {
			background-color: var(--color-foreground-light);
			border-radius: var(--border-radius-base);
			line-height: var(--font-line-height-xloose);
			padding: var(--spacing-s);
			font-size: var(--font-size-s);
			white-space: pre-wrap;
		}
	}
}
.contentText {
	padding-top: var(--spacing-s);
	font-size: var(--font-size-xs);
	// max-height: 100%;
}
.block {
	border: 1px solid var(--color-foreground-base);
	background: var(--color-background-xlight);
	padding: var(--spacing-xs);
	border-radius: 4px;
	margin-bottom: var(--spacing-2xs);
}
.blockContent {
	height: 0;
	overflow: hidden;

	&.blockContentExpanded {
		height: auto;
	}
}
.runText {
	line-height: var(--font-line-height-regular);
	white-space: pre-line;
}
.rawSwitch {
	margin-left: auto;

	& * {
		font-size: var(--font-size-2xs);
	}
}
.blockHeader {
	display: flex;
	gap: var(--spacing-xs);
	cursor: pointer;
	/* This hack is needed to make the whole surface of header clickable  */
	margin: calc(-1 * var(--spacing-xs));
	padding: var(--spacing-xs);

	& * {
		user-select: none;
	}
}
.blockTitle {
	font-size: var(--font-size-2xs);
	color: var(--color-text-dark);
}
.blockToggle {
	border: none;
	background: none;
	padding: 0;
}
</style>
