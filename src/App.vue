<script setup lang="ts">
import { BarChart, LineChart } from "echarts/charts";
import type {
  // 系列类型的定义后缀都为 SeriesOption
  BarSeriesOption,
  LineSeriesOption,
} from "echarts/charts";
import {
  // 数据集组件
  DatasetComponent,
  GridComponent,
  LegendComponent,
  TitleComponent,
  TooltipComponent,
  // 内置数据转换器组件 (filter, sort)
  TransformComponent,
} from "echarts/components";
import type {
  DatasetComponentOption,
  GridComponentOption,
  // 组件类型的定义后缀都为 ComponentOption
  TitleComponentOption,
  TooltipComponentOption,
} from "echarts/components";
import * as echarts from "echarts/core";
import type { ComposeOption } from "echarts/core";
import { LabelLayout, UniversalTransition } from "echarts/features";
import { CanvasRenderer } from "echarts/renderers";

// 通过 ComposeOption 来组合出一个只有必须组件和图表的 Option 类型
type ECOption = ComposeOption<
  | BarSeriesOption
  | LineSeriesOption
  | TitleComponentOption
  | TooltipComponentOption
  | GridComponentOption
  | DatasetComponentOption
>;

// 注册必须的组件
echarts.use([
  TitleComponent,
  TooltipComponent,
  GridComponent,
  DatasetComponent,
  TransformComponent,
  BarChart,
  LineChart,
  LabelLayout,
  LegendComponent,
  UniversalTransition,
  CanvasRenderer,
]);
import { GbfrActWs } from "gbfr-act-ws-package";
import { onMounted, ref } from "vue";

const echartsDom = ref();

const params = new URLSearchParams(window.location.search);
const valTrue = ["true", "1"];
const showTitle = valTrue.includes(params.get("title") || "true");
const theme = params.get("theme") || "dark";

const rgb = (r: number, g: number, b: number) => `rgb(${r}, ${g}, ${b})`;

const actorMap = new Map([
  ["PL0000", { name: "古兰", color: rgb(107, 211, 105) }],
  ["PL0100", { name: "姬塔", color: rgb(105, 211, 166) }],
  ["PL0200", { name: "卡塔莉娜", color: rgb(82, 126, 179) }],
  ["PL0300", { name: "拉卡姆", color: rgb(216, 87, 32) }],
  ["PL0400", { name: "伊欧", color: rgb(216, 126, 54) }],
  ["PL0500", { name: "欧根", color: rgb(121, 85, 72) }],
  ["PL0600", { name: "萝赛塔", color: rgb(188, 100, 200) }],
  ["PL0700", { name: "菲莉", color: rgb(216, 185, 40) }],
  ["PL0800", { name: "兰斯洛特", color: rgb(71, 147, 204) }],
  ["PL0900", { name: "巴恩", color: rgb(45, 55, 153) }],
  ["PL1000", { name: "珀西瓦尔", color: rgb(204, 2, 0) }],
  ["PL1100", { name: "齐格飞", color: rgb(0, 185, 247) }],
  ["PL1200", { name: "夏洛特", color: rgb(254, 179, 0) }],
  ["PL1300", { name: "尤达拉哈", color: rgb(105, 191, 211) }],
  ["PL1400", { name: "娜露梅", color: rgb(178, 33, 144) }],
  ["PL1500", { name: "冈达葛萨", color: rgb(136, 14, 79) }],
  ["PL1600", { name: "塞达", color: rgb(204, 71, 61) }],
  ["PL1800", { name: "卡莉奥丝特罗", color: rgb(255, 202, 40) }],
  ["PL1700", { name: "巴萨拉卡", color: rgb(111, 16, 165) }],
  ["PL1900", { name: "伊德", color: rgb(72, 56, 166) }],
]);

const style = {
  bleed: 8,
  backgroundColor: theme === "dark" ? "rgba(0,0,0,0.5)" : "white",
  names: {
    color: theme === "dark" ? "white" : "black",
    fontSize: "12px",
    fontFamily: "Microsoft YaHei",
    margin: 8,
  },
  damages: {
    color: theme === "dark" ? undefined : "gray",
    fontSize: "12px",
    fontFamily: "Microsoft YaHei",
    distance: 5,
  },
};

onMounted(() => {
  const gbfrActWs = new GbfrActWs({
    port: 24399,
    updateInterval: 500,
    reconnectTimeout: 3000,
    maxCombats: 10,
  });

  const chartBox = echarts.init(
    echartsDom.value,
    theme === "dark" ? "dark" : null,
    {
      renderer: "svg",
      useDirtyRect: true,
      locale: "ZH",
    },
  );

  const option: ECOption = {
    backgroundColor: style.backgroundColor,
    grid: {
      top: showTitle ? 25 : 5,
      bottom: 5,
    },
    xAxis: {
      max: "dataMax",
      axisLine: {
        show: false,
      },
      splitLine: {
        show: false,
      },
      axisLabel: {
        show: false,
      },
    },
    yAxis: {
      position: "left",
      type: "category",
      inverse: true,
      axisLine: {
        show: false,
      },
      axisTick: {
        show: false,
      },
      axisLabel: {
        show: true,
        fontWeight: "normal",
        overflow: "break",
        color: style.names.color,
        fontSize: parseInt(style.names.fontSize),
        fontFamily: style.names.fontFamily,
        margin: style.names.margin,
      },
      animationDuration: 0,
      animationDurationUpdate: 300,
    },
    series: [
      {
        realtimeSort: true,
        type: "bar",
        label: {
          show: true,
          position: "right",
          valueAnimation: true,
          formatter: (params: any): string => params.value.toLocaleString(),
          textBorderWidth: 1,
          textBorderType: "solid",
          color: style.damages.color,
          fontSize: parseInt(style.damages.fontSize),
          fontFamily: style.damages.fontFamily,
          distance: style.damages.distance,
        },
      },
    ],
    legend: {
      show: true,
    },
    animationDuration: 0,
    animationDurationUpdate: 300,
    animationEasing: "linear",
    animationEasingUpdate: "linear",
  };
  chartBox.setOption(option);

  let lastActorNames = "";

  const getTextSize = (text: string, font: { fontSize: string; fontFamily: string }) => {
    const span = document.getElementById("textSize");
    if (!span) return { width: 0, height: 0 };
    span.style.fontSize = font.fontSize;
    span.style.fontFamily = font.fontFamily;
    span.textContent = text;
    return { width: span.offsetWidth, height: span.offsetHeight };
  };

  gbfrActWs.on("combatData", (data): void => {
    // update yAxis
    if (lastActorNames !== data.actors.join("/")) {
      const names = data.actors.map((v) => {
        let name = actorMap.get(v.type.toUpperCase())?.name ?? v.hexId;
        if (
          data.actors.find(
            (d) => d.partyIdx !== v.partyIdx && d.hexId === v.hexId,
          )
        ) {
          const index = `${v.partyIdx + 1}`;
          return `[${index}]${name}`;
        }
        return name;
      });
      const left = style.names.margin + style.bleed
        + names.reduce((max, name) => Math.max(getTextSize(name, style.names).width, max), 0);
      chartBox.setOption<ECOption>({ grid: { left }, yAxis: { data: names } });
    }

    lastActorNames = data.actors.join("/");

    const right = style.damages.distance + style.bleed
      + getTextSize(data.actors.reduce((max, actor) => Math.max(max, actor.damage), 0).toLocaleString(), style.damages)
        .width;

    chartBox.setOption<ECOption>({
      grid: { right },
      title: {
        show: showTitle,
        text: `${data.duration.MMSS} - 小队秒伤：${
          Math.round(
            data.partyDamage / data.duration.seconds,
          ).toLocaleString()
        }`,
      },
      series: [
        {
          data: data.actors.map((v) => ({
            value: v.damage,
          })),
          label: {
            show: true,
          },
          itemStyle: {
            color: (params: any) => {
              try {
                const key = data.actors[params!.dataIndex].type;
                return actorMap.get(key.toUpperCase())?.color ?? "";
              } catch {
                return "";
              }
            },
          },
        },
      ],
    });
  });
  window.addEventListener("resize", function() {
    chartBox.resize();
  });
});
</script>

<template>
  <div ref="echartsDom" class="echarts"></div>
  <span id="textSize"></span>
</template>

<style scoped>
.echarts {
  width: 100vw;
  height: 100vh;
}

#textSize {
  visibility: hidden;
  white-space: nowrap;
  position: absolute;
  top: -9999px;
}
</style>
