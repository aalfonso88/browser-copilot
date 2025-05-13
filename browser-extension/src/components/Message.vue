<script lang="ts" setup>
import { computed, nextTick, ref, onMounted, onBeforeUnmount } from "vue";
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
import { AgentFlow, FlowStep } from "../scripts/flow";

const props = defineProps<{
  text: string;
  file: Record<string, string>;
  isUser: boolean;
  isComplete: boolean;
  isSuccess: boolean;
  agentLogo: string;
  agentName: string;
  agentId: string;
  steps: AgentFlow;
  isWelcome?: boolean;
}>();
const { t } = useI18n();
const renderedMsg = computed(() => (props.isUser ? props.text.replaceAll("\n", "<br/>") : renderMarkDown(props.text)));
const messageElement = ref<HTMLElement | null>(null);
const resizeObserver: ResizeObserver = new ResizeObserver(onResize);
var chart: any;

var prevWidth: number = 0;
const isExpanded = ref(false);
const showToggleButton = ref(false);

const allStepsToDisplay = computed(() => {
  const staticSteps = ["Analyzing user request", "Consulting agent"];
  const realSteps = props.steps?.steps?.map((s) => s.value?.trim()).filter((v): v is string => !!v) || [];
  const finalStep = ["Composing final response"];
  return [...staticSteps, ...realSteps, ...finalStep];
});

const visibleStepIndex = ref(0);

onMounted(() => {
  if (props.isWelcome) {
    showToggleButton.value = true;
    return;
  }
  const interval = setInterval(() => {
    if (visibleStepIndex.value < allStepsToDisplay.value.length - 1) {
      visibleStepIndex.value++;
    } else {
      clearInterval(interval);
      visibleStepIndex.value++;
      showToggleButton.value = true;
    }
  }, 1000);
});

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

    <div
      v-if="!isUser && !props?.isWelcome"
      class="w-full bg-violet-100 text-xs text-gray-700 m-1 p-2 rounded reasoning-transition"
      :class="{
        'flex flex-col justify-start items-start max-h-[400px]': isExpanded,
        'flex justify-center items-center max-h-[1.5rem] min-h-[1.5rem]': !isExpanded,
      }"
    >
      <ul class="" style="list-style: none; padding-left: 0">
        <template v-if="isExpanded">
          <li class="list-none" v-for="(step, index) in allStepsToDisplay" :key="index">
            {{ step }}
          </li>
        </template>

        <template v-else-if="visibleStepIndex < allStepsToDisplay.length">
          <li class="list-none h-[1.5rem] flex items-center justify-center w-full">
            {{ allStepsToDisplay[visibleStepIndex] }}
          </li>
        </template>
      </ul>

      <div v-if="showToggleButton" class="w-full flex justify-center h-[1.5rem]">
        <button
          @click="isExpanded = !isExpanded"
          class="h-full w-full flex justify-center items-center text-xs text-gray-600 hover:text-violet-500 transition"
        >
          <component :is="isExpanded ? ChevronUpIcon : ChevronDownIcon" class="w-5 h-5" />
        </button>
      </div>
    </div>

    <div class="mt-2 ml-8 mr-2" v-if="isUser || (!isUser && showToggleButton)">
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

.reasoning-transition {
  overflow: hidden;
  transition: all 0.5s ease-in-out;
}

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
