<script lang="ts" setup>
import { computed, nextTick, ref, onMounted, onBeforeUnmount, watch } from "vue";
import { useI18n } from "vue-i18n";
import MarkdownIt from "markdown-it";
import hljs from "highlight.js";
import "highlight.js/styles/base16/gigavolt.min.css";
import MarkdownItPlantuml from "markdown-it-plantuml";
import { ExclamationCircleIcon, CircleFilledIcon, ChevronDownIcon, ChevronUpIcon } from "vue-tabler-icons";
import NewPromptButton from "./NewPromptButton.vue";
import CopyButton from "./CopyButton.vue";
import * as echarts from "echarts";
import moment from "moment";
import { AgentFlow } from "../scripts/flow";

const props = defineProps<{
  text: string;
  file: Record<string, string>;
  isUser: boolean;
  isComplete: boolean;
  isSuccess: boolean;
  agentLogo: string;
  agentName: string;
  agentId: string;
  onCancelMessage: () => Promise<void>;
  steps: AgentFlow;
  isWelcome?: boolean;
}>();
const { t } = useI18n();
const renderedMsg = computed(() => (props.isUser ? props.text.replaceAll("\n", "<br/>") : renderMarkDown(props.text)));
const messageElement = ref<HTMLElement | null>(null);
const resizeObserver: ResizeObserver = new ResizeObserver(onResize);
var chart: any;

var prevWidth: number = 0;

const panelExpanded = ref(true);
function togglePanel() {
  panelExpanded.value = !panelExpanded.value;
}

const showFinalMessage = ref(false);
watch(
  () => props.steps.steps.at(-1)?.action,
  (lastAction) => {
    if (lastAction === "end" && !showFinalMessage.value) {
      setTimeout(() => {
        showFinalMessage.value = true;
      }, 1500);
    }
  }
);

const cancelRequested = ref(false);
const cancelState = ref<"active" | "cancelled">("active");

async function cancelResponse() {
  cancelRequested.value = true;
  cancelState.value = "cancelled";
  try {
    await props.onCancelMessage();
  } catch (err) {}
}

function renderMarkDown(text: string) {
  let md = new MarkdownIt({
    highlight: (code: string, lang: string) => {
      let ret = code;
      if (lang && hljs.getLanguage(lang)) {
        try {
          ret = hljs.highlight(code, { language: lang }).value;
        } catch (__) {}
      }
      return '<pre><code class="hljs">' + ret + "</code></pre>";
    },
  });
  useTargetBlankLinks(md);
  useEcharts(md);
  md.use(MarkdownItPlantuml);
  return md.render(text);
}

function useTargetBlankLinks(md: MarkdownIt) {
  let defaultRender =
    md.renderer.rules.link_open ||
    function (tokens, idx, options, env, self) {
      return self.renderToken(tokens, idx, options);
    };
  md.renderer.rules.link_open = function (tokens, idx, options, env, self) {
    tokens[idx].attrSet("target", "_blank");
    return defaultRender(tokens, idx, options, env, self);
  };
}

function useEcharts(md: MarkdownIt) {
  const defaultRender =
    md.renderer.rules.fence ||
    function (tokens, idx, options, env, self) {
      return self.renderToken(tokens, idx, options);
    };
  md.renderer.rules.fence = function (tokens, idx, options, env, self) {
    const token = tokens[idx];
    const code = token.content.trim();
    if (token.info === "echarts") {
      nextTick().then(() => {
        const container = messageElement.value!;
        const chartDiv = container.querySelector(".echarts");
        const chartData = container.querySelector(".echarts-data");
        if (chartDiv && chartData) {
          chart = echarts.init(chartDiv as HTMLDivElement);
          const options = JSON.parse(chartData.textContent || "");
          solveEchartsFormatter(options.xAxis.axisLabel);
          solveEchartsFormatter(options.xAxis.axisPointer.label);
          chart.setOption(options);
        }
      });
      return `<div class="echarts" style="width: 100%; height: 200px;"></div><div class="echarts-data" style='display:none'>${code}</div>`;
    }
    return defaultRender(tokens, idx, options, env, self);
  };
}

function solveEchartsFormatter(obj: any) {
  if (obj && obj.formatter) {
    if (obj.formatter.name === "formatEpoch") {
      obj.formatter = formatEpoch(obj.formatter);
    }
  }
}

function formatEpoch(config: any): (value: any) => string {
  return (value: any) => {
    if (typeof value === "object") {
      value = value.value;
    }
    const time = moment(parseInt(value));
    return time.format(config.format);
  };
}

function onResize() {
  if (messageElement.value!.scrollWidth != prevWidth && chart) {
    prevWidth = messageElement.value!.scrollWidth;
    chart.resize();
  }
}

onMounted(() => {
  if (messageElement.value) {
    resizeObserver.observe(messageElement.value);
  }
});

onBeforeUnmount(() => {
  if (messageElement.value) {
    resizeObserver.unobserve(messageElement.value);
  }
});
</script>

<template>
  <div class="flex flex-col mb-1 p-1 min-w-7" :class="!isSuccess ? ['border-red-500', 'border-b'] : []">
    <div class="flex items-center flex-row">
      <template v-if="isUser">
        <circle-filled-icon class="text-violet-600" />
      </template>

      <template v-else-if="!isUser && isSuccess">
        <img :src="agentLogo" class="w-5 mr-1 rounded-full" />
      </template>
      <template v-else>
        <exclamation-circle-icon class="text-red-600" />
      </template>

      <span class="text-base">{{ isUser ? t("you") : agentName }}</span>

      <div class="flex-auto flex justify-end">
        <CopyButton v-if="!isUser && text" :text="text" :html="renderedMsg" />
        <NewPromptButton v-if="isUser && text" :is-large-icon="false" :text="text" :agent-id="agentId" />
      </div>
    </div>

    <div v-if="!isUser && !isWelcome" class="w-full bg-violet-100 text-xs text-gray-700 m-2 p-2 rounded">
      <div
        :class="[
          'relative overflow-hidden px-2 w-full text-xs text-gray-700 bg-violet-100 rounded',
          panelExpanded ? 'max-h-[800px] pt-2 pb-2' : 'max-h-[28px]',
          'transition-[max-height] duration-1000 ease-in-out',
        ]"
      >
        <ul v-show="panelExpanded" class="w-full list-none pl-0">
          <li v-for="(step, index) in steps.steps" :key="index" class="list-none">
            {{ step.value }}
          </li>
        </ul>

        <div v-if="!showFinalMessage" class="absolute top-1 right-2 z-10">
          <button
            @click="cancelResponse"
            :disabled="cancelState === 'cancelled'"
            class="text-xs px-3 py-1 border rounded transition"
            :class="
              cancelState === 'cancelled'
                ? 'border-gray-400 text-gray-400 cursor-not-allowed'
                : 'border-red-500 text-red-500 hover:underline'
            "
          >
            {{ cancelState === "cancelled" ? "Cancelado" : "Cancel" }}
          </button>
        </div>

        <div v-if="showFinalMessage" class="w-full flex justify-center mt-1">
          <button @click="togglePanel" class="text-xs text-gray-600 hover:text-violet-500 transition -mt-2">
            <component :is="panelExpanded ? ChevronUpIcon : ChevronDownIcon" class="w-5 h-5" />
          </button>
        </div>
      </div>
    </div>

    <div v-if="isWelcome || isUser || (!isUser && showFinalMessage)" class="mt-2 ml-8 mr-2">
      <div>
        <template v-if="file.data">
          <audio controls>
            <source :src="file.url" type="audio/webm" />
          </audio>
        </template>
        <template v-if="text">
          <div
            v-html="renderedMsg"
            ref="messageElement"
            class="flex flex-col text-sm font-light leading-tight gap-2 rendered-msg"
          />
        </template>
      </div>
      <div class="ml-3 dot-pulse" v-if="!isComplete" />
    </div>
  </div>
</template>
<style lang="scss">
@use "three-dots" with (
  $dot-width: 5px,
  $dot-height: 5px,
  $dot-color: var(--accent-color)
);

.rendered-msg pre {
  padding: 15px;
  background: #202126;
  border-radius: 8px;
  text-wrap: wrap;
}

// Fix: Inadequate gap between code blocks within list items.
.rendered-msg li pre {
  margin-bottom: 10px;
}

.rendered-msg pre {
  box-shadow: var(--shadow);
}

.rendered-msg pre code.hljs {
  padding: 0px;
}

div a {
  color: var(--accent-color);
  text-decoration: none;
}

.rendered-msg table {
  width: 100%;
  box-shadow: var(--shadow);
}

.rendered-msg thead tr {
  background-color: #ece6f5;
}

.rendered-msg th,
.rendered-msg td {
  padding: var(--half-spacing);
  border: var(--border);
}

.rendered-msg tbody tr:hover {
  background-color: #f1f1f1;
}

.echarts {
  box-shadow: var(--shadow);
  border-radius: var(--spacing);
  width: 100%;
  padding: var(--half-spacing);
}

.rendered-msg > img {
  box-shadow: var(--shadow);
  border-radius: var(--spacing);
  width: fit-content;
}
</style>

<i18n>
{
  "en": {
    "you": "You"
  },
  "es": {
    "you": "Tú"
  }
}
</i18n>
