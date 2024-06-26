<template>
	<Teleport :to="container">
		<div
			ref="videoStatsRef"
			class="seventv-video-stats"
			@click="openStatsOverlay"
			@mouseenter="videoStatsTooltip.show(videoStatsRef)"
			@mouseleave="videoStatsTooltip.hide()"
		>
			<figure v-if="videoStats.playbackRate > 1">
				<ForwardIcon />
			</figure>
			<figure v-else>
				<GaugeIcon />
			</figure>
			<span>{{ latency }}s</span>
		</div>
	</Teleport>
</template>

<script setup lang="ts">
import { onMounted, onUnmounted, reactive, ref, watch } from "vue";
import { HookedInstance } from "@/common/ReactHooks";
import { definePropertyHook, unsetPropertyHook } from "@/common/Reflection";
import { useTooltip } from "@/composable/useTooltip";
import ForwardIcon from "@/assets/svg/icons/ForwardIcon.vue";
import GaugeIcon from "@/assets/svg/icons/GaugeIcon.vue";
import PlayerStatsTooltip from "./PlayerStatsTooltip.vue";

const props = defineProps<{
	advancedControls: HookedInstance<Twitch.MediaPlayerAdvancedControls>;
	mediaPlayer?: Twitch.MediaPlayerInstance;
}>();

const latency = ref<string>("-.--");
const videoStats = reactive({
	droppedFrames: 0,
	playbackRate: 0,
	bitrate: "0",
	width: 0,
	height: 0,
	framerate: 0,
	bufferSize: 0,
});

// Container element for the video stats hoverable icon
const container = document.createElement("seventv-container");

const videoStatsRef = ref();
const videoStatsTooltip = useTooltip(PlayerStatsTooltip, videoStats);

function openStatsOverlay() {
	const controls = props.advancedControls.component;
	const isOpen = document.querySelector("[data-a-target='player-overlay-video-stats']");
	controls.setStatsOverlay(isOpen ? 0 : 1);
}

function remount() {
	const sibling = document.querySelector<HTMLElement>("span.live-time")?.parentElement;
	if (!sibling || sibling.parentElement?.contains(container)) return;

	sibling.insertAdjacentElement("afterend", container);
}

defineExpose({ remount });

watch(
	latency,
	() => {
		if (!props.mediaPlayer) return;

		videoStats.droppedFrames = props.mediaPlayer.getDroppedFrames();
		videoStats.bitrate = (props.mediaPlayer.getVideoBitRate() / 1000)?.toFixed(0) ?? "0";
		videoStats.framerate = props.mediaPlayer.getVideoFrameRate();
		videoStats.playbackRate = props.mediaPlayer.getPlaybackRate();
		videoStats.width = props.mediaPlayer.getVideoWidth();
		videoStats.height = props.mediaPlayer.getVideoHeight();
		videoStats.bufferSize = props.mediaPlayer.getBufferDuration();
	},
	{ immediate: true },
);

watch(
	() => props.mediaPlayer,
	(mediaPlayer, old) => {
		if (old && mediaPlayer !== old) {
			unsetPropertyHook(old.core.state, "liveLatency");
		} else {
			if (!mediaPlayer) return;
			definePropertyHook(mediaPlayer.core.state, "liveLatency", {
				value: (v: number) => {
					latency.value = v?.toFixed(2) ?? "-.--";
				},
			});
		}
	},
	{ immediate: true },
);

onMounted(() => {
	remount();
});

onUnmounted(() => {
	container.remove();
});
</script>

<style scoped lang="scss">
.seventv-video-stats {
	margin-right: 0.5rem;
	display: grid;
	grid-template-columns: auto 1fr;
	column-gap: 0.5rem;
	padding: 0.25rem 0.5rem;
	font-family: "Helvetica Neue", sans-serif;
	font-variant-numeric: tabular-nums;
	cursor: pointer;

	&:hover {
		background: hsla(0deg, 0%, 30%, 32%);
	}
}

.seventv-video-stats > figure {
	display: grid;
	place-items: center;
}
</style>
