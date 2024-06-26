<template>
	<template v-if="ch?.instances">
		<template v-for="i of ch.instances" :key="i">
			<template v-for="cmd of commands" :key="cmd">
				<component :is="cmd" v-bind="{ add: i.component.addCommand, remove: i.component.removeCommand }" />
			</template>
		</template>
	</template>
</template>
<script setup lang="ts">
import type { Component } from "vue";
import { until } from "@vueuse/core";
import { useComponentHook } from "@/common/ReactHooks";
import { declareModule } from "@/composable/useModule";
import Dashboard from "./Commands/Dashboard.vue";
import Nuke from "./Commands/Nuke.vue";
import Reresh from "./Commands/Refresh.vue";
import Song from "./Commands/Song.vue";

const { dependenciesMet, markAsReady } = declareModule("command-manager", {
	name: "Command Manager",
	depends_on: ["chat", "chat-input"],
});

let ch: ReturnType<typeof useComponentHook<Twitch.ChatCommandComponent>> | undefined = undefined;

defineExpose({
	addCommand: (c: Twitch.ChatCommand) => ch?.instances[0]?.component.addCommand(c),
	removeCommand: (c: Twitch.ChatCommand) => ch?.instances[0]?.component.removeCommand(c),
});

await until(dependenciesMet).toBe(true);

const commands = [Dashboard, Song, Reresh, Nuke] as Component[];

useComponentHook<Twitch.ChatCommandGrouperComponent>(
	{
		parentSelector: ".chat-input__textarea",
		predicate: (n) => n.determineGroup,
		maxDepth: 50,
	},
	{
		functionHooks: {
			determineGroup(this, old, command) {
				return command.group ? command.group : old.call(this, command) ?? "Twitch";
			},
		},
	},
);

ch = useComponentHook<Twitch.ChatCommandComponent>({
	childSelector: ".stream-chat",
	predicate: (n) => n.addCommand,
	maxDepth: 50,
});
markAsReady();
</script>
